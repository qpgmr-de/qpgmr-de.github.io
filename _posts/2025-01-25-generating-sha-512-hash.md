## Generating SHA-512 hashes using IBM i crypto APIs

My task was, to find differences between many source code files (members or IFS) on different systems. To achieve this, I settled on the idea, to build hash codes over the sources, and to compare the hashes. And calculating the hash values was easier as I initially thought, because IBM i has a great set of integrated crypto API’s.

I don’t want to go through the source code line by line, as I think you are a seasoned ILE-RPG programmer. But I want to go describe the prototypes of the APIs that we use.

IBM has created some copy/include files with data structure definitions for the APIs, so you have to include them.

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-01.rpgle"></script>

But we won’t use these data structures directly, because they are not qualified, and I don’t like unqualified data structures, so we will use them as templates for likeds definitions.

### Qc3CreateAlgorithmContext API

So here is a prototype for the first API – before you can use any crypto API you have to create a „context token“ – which is some kind of „pointer“ or „session id“ for your calculations. You can open multiple context tokens in parallel, so you can also run multiple crypto algorithms in parallel, each with its own context token.

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-02.rpgle"></script>

The context token is returned in crypto_context_token, whereas any errors are reported in the data structure usec.

The QC3D0500 data structure looks fairly simple – but you don’t have to code it, you can simply use likeds(QC3D0500) and you will get a data structure like this:

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-03.rpgle"></script>

The only subfield qc3ha identifies the hashing algorithm that should be used, and the following values are used to select it:

  1 = MD5
  2 = SHA-1
  3 = SHA-256
  4 = SHA-384
  5 = SHA-512
  7 = SHA-224
  
Of course the crypto APIs can do a lot more, than only generating hashes. DES, AES or RSA – you can do a lot with these APIs – but you will have to work through the docs.

### Qc3CalculateHash API

The prototype of Qc3CalculateHash looks a bit mot complicated.

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-04.rpgle"></script>

There are different ways to supply the input data that should be hashed – the most simple way is, to supply a (char)pointer to the data to be hashed, the length of this data chunk and the value `DATA0100` as the the format of the input data.

The algorithm description data structure is also not so complicated – as always simply use likeds(QC3D0100) to declare your own qualified data structure:

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-05.rpgle"></script>

The first subfield qc3act holds the algorithm context token, that we receive from our call to Qc3CreateAlgorithmContext.

The second subfield qc3fof is the „final operation flag“. A value of ‚0‘ means, that there are more API calls with more data are coming to generate the hash. A value of ‚1‘ means, that the API should finally generate the hash. If you have to hash only one block of data, you cann just call the API once, with qc3fof set to ‚1‘.

### cvthc „API“

The last API we will use is the cvthc API. This is no „normal“ API – even the very short name doesn’t sound like one. cvthc is a machine interface (MI) instruction, but it can be called just like a normal API.

The notation of MI instructions is sometimes a bit strange, because cvthc stands for „convert to hex from char“.

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-06.rpgle"></script>

The cvthc instruction simply converts a binary block of data to a hex representation – so it converts a byte with a value of 0xF1 (decimal 241, binary 11110001 or an EBCDIC char ‚1‘) into a string with a value of 'F1'.

This means, that a block of 512 bits (thats 64 bytes) is converted to a string of 128 chars.

I chose the char representation of the hash, because I store it in a database, and this is easier with a string, than with a binary field.

### The main program

The main program looks very simple in fact:

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-07.rpgle"></script>

I don’t think, that I have to walk thru that code.

### The GetSourceHash procedure

We have our procedure (or function) which takes a varchar(1000) string with the IFS path to the source file or member as a parameter, and returns a varchar(128) string with the hash value in hex notation.

We need some system API includes – the qc2cci member contains the declarations for the crypto APIs, and the qusec member contains the usual error/response data structure for nearly all system APIs.

I also declare some constants and some variables – pretty straight forward.

The first API we need is Qc3CreateAlgorithmContext – it creates a new „context“ to run one of the crypto algorithms. Which algorithm should be used, is configured in the ALGD0500 format.

The next API is Qc3CalculateHash – this one really calculates the hash value of a given set of data. It can be used with different configurations – single shot (e.g. for password hashing) or like we will be using it, in consecutive calls to input data (like the lines of source file) and returning the hash value at the end.

With the parameters crypto_prov and crypto_dev you could even select a dedicated crypto processor, which is installed in your machine – we don’t have one, so we tell the OS to select any crypto solution it likes – if no dedicated hardware is available, this means software.

And of course we need some variables – one holds the binary hash value of 512 bits – this means 64 bytes. The other holds that hash value translated into „readable“ hex notation – this means, that a one-byte value of 0xF1 is translated to a string with two chars „F1“. It also means, that we need double the storage, but strings like there are easier to store.

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-08.rpgle"></script>

We have to tell the SQL processor how we want to handle warnings and errors – I always like to handle them myself, so „continue“ is my favorite,

And now that we declared everything, we start to read through the IFS file with the SQL table function ifs_read – and the best, this function also allows to read source members, simply by using the IFS notation of the member, like in the main procedure. No extra work for us.

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-09.rpgle"></script>

We have opened the file or member, now initialize the crypto APIs and get a crypto context, which we need for the the next API. And we also select, that we want to make consecutive calls to the next API.

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-10.rpgle"></script>

We loop through the lines of the file or member – ifs_read does remove CR and/or LF chars at the end of the line, so we really only get the data of each line. And it also translates the data according to the CCSID.

In an IFS file, all lines have different line lengths, because trailing blanks are normally removed. This is not the case with source members. So to compare only the „real source code“ without any trailing blanks, we like to remove them – so that the source data from an IFS file, is the same as the source data from a source member.

But it means, that an empty source code line will result in a data chuck with a length of zero – and this cannot be processed by the API, so any completely empty line, will be translated to one space char.

If you step through the program with debug, you will notice, that the „hashBin“ parameter of the API won’t receive a value during the calls inside the loop. This is because, we do the consecutive calls and receive the hash value later.

<script src="https://gist.github.com/qpgmr-de/2b6ceccc7fe646a0ac571ba5733d626f.js?file=generating-sha-512-hash-11.rpgle"></script>

We made it – all source lines were fetched – now we receive the hash value. To do that, we have to set the qc3fof field of the ALGD0100 data structure to „1“ and call the API without data.

In the last step, we use our MI instruction to convert the 512-bit (64 byte) binary hash value to a hex string, and finally we return it to the caller.

I think, this is a good example how to use IBM i system APIs and MI instructions, and I hope you enjoyed the post.
