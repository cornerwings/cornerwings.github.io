---
layout: post
title:  "Interfacing with native methods on Graal VM"
date:   2018-07-03 16:20:04 -0700
---

Java has quite possibly one of the worst ways to interact with native methods using JNI (Java Native Interface). JNI APIs follow weird conventions and the language boundary interop is unbelievably cumbersome to deal with. Graal VM introduces a new way to cross language boundary that is easier to maintain and allows interop with any native library.

Exchanging anything other than primitive data via JNI typically involves either reaching back to the VM (very slow) or use a serialization library like Protobuf (still slow) or directly access memory by address in Java via sun.misc.Unsafe (fast but unstable). That's why I was super excited for [Graal VM's polyglot][graal-polyglot] abilities which will essentially eliminate the need for JNI and introduce a safer way to cross the language boundary from Java. LLVM interoperability in [reference manual][graal-llvm-interop] gave me a lot of hope and showed how simple this can be.

Turns out there are still some limitations in polyglot capabilities of Graal VM. For Java interop with native methods it requires you to create a native image (compile the java application to executable code).

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">The direct access to C data structures via interoperability is currently only supported for JavaScript, Ruby, R, or Python (see <a href="https://t.co/chX23U78kA">https://t.co/chX23U78kA</a> for an example). How to interact between Java and C/C++ when creating a native image is described here: <a href="https://t.co/HeuOIDOlOq">https://t.co/HeuOIDOlOq</a></p>&mdash; Thomas Wuerthinger (@thomaswue) <a href="https://twitter.com/thomaswue/status/992798186543702016?ref_src=twsrc%5Etfw">May 5, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

The only problem is that native image building functionality in Graal VM is quite limited (as of now) and it is not always possible to build one for complex Java applications. I still wanted to see if this is indeed simpler than using JNI and if there is any performance gap between these approaches (the fastest way that I know of is to expose the address via JNI method and write wrapper classes that access memory directly using Unsafe in Java).

## Exposing C data structures
Although not quite as neat as exposing these structs to dynamic languages like JavaScript etc., it is still quite convenient in Graal VM. Firstly there is no need to implement additional APIs just for the sake of language interop and you can simply integrate with your existing headers and library files. This means that it is possible to integrate with any arbitrary library provided that initial mapping of structs is done.

Let's start with a simple data structure (in file ```triple.h```) and create the corresponding structure for that in Java,
{% highlight c %}
typedef struct value_t {
    int type;
    long id;
} value_t;

typedef struct triple_t {
    value_t subject;
    value_t predicate;
    value_t object;
} triple_t;
{% endhighlight %}

C struct package provides us with required functionality to achieve this. For our example, we need to use two concepts to complete the mapping,
* [CField][graal-api-cfield] - Maps a primitive field in struct to Java.
* [CFieldAddress][graal-api-cfield-address] - Maps a complex (or nested) struct by leveraging composition in Java.

Mapping ```value_t``` is pretty straightforward and ```CField``` lets us create getters and setters that directly interact with native memory.

{% highlight java %}
@CStruct("value_t")
interface Value extends PointerBase {

    @CField("type")
    int getDataType();

    @CField("type")
    void setDataType(int type);

    @CField("id")
    long getId();

    @CField("id")
    void setId(long id);

}
{% endhighlight %}

That’s not so bad - no hardcoded constants or crazy long method names and it allows bi-directional mutations. This is already better than JNI. Similarly, we can map Triple which is a composite structure that carries 3 values. A slight difference here is that we will reference the Java ```Value``` interface using address indirection. This means we don’t get setters/getters like the previous example which makes sense. You cannot really set a new subject to the C struct as it is defined (it is possible if those member fields are pointers to ```value_t``` instead).

{% highlight java %}
@CStruct("triple_t")
interface Triple extends PointerBase {

    @CFieldAddress("subject")
    Value subject();

    @CFieldAddress("predicate")
    Value predicate();

    @CFieldAddress("object")
    Value object();

}
{% endhighlight %}

Voila! We can now access these just like rest of Java code. 

{% highlight java %}
// Somehow get a triple from native code
Triple triple = ...;
long subjectId = triple.subject().getId();
{% endhighlight %}

I glossed over one thing to get to mapping part but Graal VM requires you to enclose these struct classes in some [CContext][graal-api-ccontext]. This is used as part of native-image build to resolve the offsets correctly.

{% highlight java %}
@CContext(Headers.class)
public class Example {

    static class Headers implements CContext.Directives {
        @Override
        public List<String> getHeaderFiles() {
            return Collections.singletonList("\"/full/path/to/triple.h\"");
        }
    }

    @CStruct("value_t")
    interface Value extends PointerBase { ... }

    @CStruct("triple_t")
    interface Triple extends PointerBase { ... }

}
{% endhighlight %}

## Interacting with native functions
Above section showed how C data structures can be exposed to Java. But we still need some functions to create these structures so rest of code can interact with them. One thing I like about this is that the object/struct creation is completely managed by native code, unlike JNI where there is no such strict contract (I may be wrong but couldn’t find any documentation that says otherwise).

Let's say we have this basic function which creates the triple and returns the address to the created object,
{% highlight c %}
triple_t* allocRandomTriple() {
    triple_t *triple = (triple_t*) malloc(sizeof(triple_t));
    triple->subject.id = 1;
    triple->predicate.id = 2;
    triple->object.id = 3;
    return triple;
}
{% endhighlight %}

Calling this from Java is really simple with just one annotation and reference to ```Triple``` class created above.
{% highlight java %}
@CFunction(transition = Transition.NO_TRANSITION)
public static native Triple allocRandomTriple();
{% endhighlight %}

After that Graal still needs to be informed which library provides this method and this can be done with another annotation (assuming the shared library built with above called is named ```libtriple.so```).
{% highlight java %}
@CContext(Headers.class)
@CLibrary("triple")
public class Example { ... }
{% endhighlight %}

```Transition``` mentioned above can be further tuned to improve the performance. Using the default value for transition causes Graal to transition the thread state from Java to C and it will make Java parts of stack available. No transition can be used in more tighter loops where the native method is just doing some raw computation.

## Native interaction via JNI/Unsafe
Here is an example of creating a similar interface using ```sun.misc.Unsafe``` and getting access to the native data structure in Java. Not only this is very error-prone and hard to debug but extremely hard to evolve as structure evolves in the development lifecycle (not to mention the deprecation of Unsafe in recent JDK versions). A safer alternative is to create ```DirectByteBuffer``` via JNI API and use that to achieve similar functionality (with a small performance penalty since it involves reaching back to VM from native context).

First we need to create a special JNI method that allows Java to bind native methods. JNI methods need to follow this specific convention ```Java_package_name_ClassName_method``` for them to be detected correctly.
{% highlight c %}
JNIEXPORT jlong JNICALL Java_NativeTriple_allocRandomTriple(JNIEnv *env, jclass clazz) {
    triple_t *triple = allocRandomTriple();
    return (long) triple;
}
{% endhighlight %}

Basically, all we are doing with the native method is to expose the raw memory address to Java and we use Unsafe APIs to access the memory directly. Obviously, this is very unsafe so we need to hide this abomination from callers, here’s an example of a wrapper class that hides these details (still a major pain to maintain the magic numbers on an ongoing basis).

{% highlight java %}
public class NativeTriple {

    private long ptr;
    private NativeValue subject;
    private NativeValue predicate;
    private NativeValue object;

    private NativeTriple(long ptr) {
        this.ptr = ptr;
        this.subject = new NativeValue(ptr + 0);
        this.predicate = new NativeValue(ptr + 16);
        this.object = new NativeValue(ptr + 32);
    }

    public static class NativeValue {

        private long ptr;
        private static Unsafe unsafe = getUnsafe();

        public NativeValue(long ptr) {
            this.ptr = ptr;
        }

        public int getType() {
            return unsafe.getInt(ptr);
        }

        public long getId() {
            return unsafe.getLong(ptr + 8);
        }

    }

    public static NativeTriple create() {
        long ptr = allocRandomTriple();
        return new NativeTriple(ptr);
    }

    private static native long allocRandomTriple();    
}
{% endhighlight %}

If you need to abstract this out further, take a look at [Javolution][javolution] codebase. Its a great reference to build a generic framework that can eliminate much of the constants in above code.

## Benchmarks
>> As always take these micro benchmarks with a grain of salt and make sure to conduct a similar experiment for your application to really quantify the difference.

Linked source code below has a very basic benchmark that runs 100M iterations interacting with these methods.
{% highlight java %}
long iterations = 100_000_000L;
long start = System.currentTimeMillis();
long sum = 0;

for (long i = 0; i < iterations; i++) {
    Triple triple = allocRandomTriple();
    sum += triple.subject().getId() + triple.predicate().getId() + triple.object().getId();
    freeTriple(triple);
}

long end = System.currentTimeMillis();
double timeTaken = ((double) (end - start) * 1_000_000L)/iterations;
{% endhighlight %}

Here are the results,

| Method      | Time per iteration |
| ------------- |-------------|
| graal/native-image      | 76.11 ns |
| jni/unsafe      | 109.42 ns      |


Not only code is cleaner with Graal VM but it is about 1.4x faster over the fastest possible JNI implementation. I am not sure how much of that is due to AOT complication or something fundamental with native method interaction. 

## Analysis
At this point, you might be wondering why even bother crossing the language boundary as performance differences in modern languages are quite small. For a database it is beneficial to handle the request/responses in Java, this lets us use server frameworks like Netty or support a variety of serialization formats/charset encodings. At the same time database is all about control, you want to make sure resources are released when they are supposed to be released and query limits are enforced strictly (memory or time). Having a fast maintainable way to cross the language boundary when needed makes these type of uses cases possible.

Even though it is required to go through the native image route in Graal VM, the actual interaction between Java and native methods is much easier/succinct compared to using JNI. With sufficient care, it could even be backward compatible as long as new members to structs are additive. I am hopeful with existing limitations of native image removed, language interop becomes straightforward and open up a new class of applicatons on Graal VM.

## Source code
All of the code mentioned in this post is available at the following repository: [graal-native-interaction][graal-native-interaction].

[graal-polyglot]: https://www.graalvm.org/docs/reference-manual/polyglot/
[graal-llvm-interop]: https://www.graalvm.org/docs/reference-manual/languages/llvm/#interoperability
[graal-api-cfield]: http://www.graalvm.org/sdk/javadoc/org/graalvm/nativeimage/c/struct/CField.html
[graal-api-cfield-address]: http://www.graalvm.org/sdk/javadoc/org/graalvm/nativeimage/c/struct/CFieldAddress.html
[graal-api-ccontext]: http://www.graalvm.org/sdk/javadoc/org/graalvm/nativeimage/c/CContext.html
[javolution]: https://github.com/javolution/javolution/tree/master/src/main/java/org/javolution/io
[graal-native-interaction]: https://github.com/cornerwings/graal-native-interaction


