## Using Java objects and methods from RPG

The possibility to create and use Java objects from RPG is nothing new - at
least I found it even in IBM i 7.2. But the documentation and the example 
source code is still in fix format RPG. 

So when I had to do split multi-page PDF files into one page per PDF files,
I quickly turned to [Apache PDFBox®](https://pdfbox.apache.org) as it is
a very nice library, which

It was quite easy, to find the necessary information in the Info Center:

- https://www.ibm.com/docs/en/i/7.5?topic=world-rpg-java

Sadly all examples are in the old fixed-form/FREE source code format. Many
examples use a copy-file [`JAVAUTIL`](https://www.ibm.com/docs/en/i/7.5?topic=java-additional-rpg-coding-using) 
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


My task was, to find differences between many source code files (members or IFS) on different systems. To achieve this, I settled on the idea, to build hash codes over the sources, and to compare the hashes. And calculating the hash values was easier as I initially thought, because IBM i has a great set of integrated crypto API’s.

I don’t want to go through the source code line by line, as I think you are a seasoned ILE-RPG programmer. But I want to go describe the prototypes of the APIs that we use.

IBM has created some copy/include files with data structure definitions for the APIs, so you have to include them.

```rpgle
/copy qsysinc/qrpglesrc,qc3cci
/copy qsysinc/qrpglesrc,qusec
```

But we won’t use these data structures directly, because they are not qualified, and I don’t like unqualified data structures, so we will use them as templates for likeds definitions.

### Qc3CreateAlgorithmContext API
