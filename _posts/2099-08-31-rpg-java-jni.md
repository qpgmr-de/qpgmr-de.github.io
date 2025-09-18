## Using Java objects and methods from RPG

The possibility to create and use Java objects from RPG is nothing new - at
least I found it even in IBM i 7.2. But the documentation and the example 
source code is still in fix format RPG. 

So when I had to create a solution, to split multi-page PDF files into one 
page per PDF files, I quickly turned to Apache PDFBox® [^1] - a very versatile
library, which makes all kinds of operations with PDF files.
[^1]: https://pdfbox.apache.org

It was quite easy, to find the necessary information in the Info Center [^2] - but sadly all examples are in the old fixed-form/FREE source code format.
[^2]: https://www.ibm.com/docs/en/i/7.5?topic=world-rpg-java

Many of the examples use a copy-file `JAVAUTIL`[^3] that makes is easy 
[^3]: https://www.ibm.com/docs/en/i/7.5?topic=java-additional-rpg-coding-using

but the copy-file and its
sevice program is only shown in fragments

which defines some [JNI (Java Native Interface)](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/)
prototypes. You can download the original here:


- https://www.ibm.com/support/pages/jni-utilities-ile-rpg-programmers-guide

But I have converted all prototypes to full-free/**free source format, and
prefixed them, so they are "thematically" bundled. 

### What you need to know upfront

Whenever your program or service program is using Java objects and calls Java 
methods, you have to code the `THREAD` keyword in your control options. Many
RPG programs rely on static (global) storage, so it's always safe to use the
`*SERIALIZE` option. But if your program is thread-safe, you might want to
set it to `*CONCURRENT`. But you shouldn't leave the `THREAD` out of your
program, as the results are unpredictable.

You might also want to include the `JAVAUTIL` copy-file and bind to the 
`JAVAUTIL` service program (or better binder directory).

```rpgle
ctl-opt thread(*serialize);
ctl-opt bnddir('JAVAUTIL');
/include javautil
```


https://www.ibm.com/docs/en/i/7.5?topic=java-additional-rpg-coding-using


