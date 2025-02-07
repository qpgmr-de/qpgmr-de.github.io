## Generating SHA-512 hashes using IBM i crypto APIs

My task was, to find differences between many source code files (members or IFS) on different systems. To achieve this, I settled on the idea, to build hash codes over the sources, and to compare the hashes. And calculating the hash values was easier as I initially thought, because IBM i has a great set of integrated crypto API’s.

I don’t want to go through the source code line by line, as I think you are a seasoned ILE-RPG programmer. But I want to go describe the prototypes of the APIs that we use.

IBM has created some copy/include files with data structure definitions for the APIs, so you have to include them.

```rpgle
/copy qsysinc/qrpglesrc,qc3cci
/copy qsysinc/qrpglesrc,qusec
```

But we won’t use these data structures directly, because they are not qualified, and I don’t like unqualified data structures, so we will use them as templates for likeds definitions.

### Qc3CreateAlgorithmContext API

So here is a prototype for the first API – before you can use any crypto API you have to create a „context token“ – which is some kind of „pointer“ or „session id“ for your calculations. You can open multiple context tokens in parallel, so you can also run multiple crypto algorithms in parallel, each with its own context token.

```rpgle
dcl-pr initCryptoAPI extproc('Qc3CreateAlgorithmContext');
  hash_algorithm likeds(QC3D0500) const;
  hash_algorithm_format char(8) const;
  crypto_context_token char(8);
  usec likeds(QUSEC);
end-pr;

dcl-ds dsAlgd0500 likeds(QC3D0500) inz;
```

The context token is returned in crypto_context_token, whereas any errors are reported in the data structure usec.

The QC3D0500 data structure looks fairly simple – but you don’t have to code it, you can simply use likeds(QC3D0500) and you will get a data structure like this:

```rpgle
dcl-ds QC3D0500 qualified;
  qc3ha int(10:0);
end-pr;
```

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

```rpgle
dcl-pr CalculateHash extproc('Qc3CalculateHash');
  input_data char(65535) const options(*varsize);
  input_data_len int(10:0) const;
  input_data_fmt char(8) const;            
  alg_desc likeds(QC3D0100) const;
  alg_desc_fmt char(8) const;            
  crypto_prov char(1) const;            
  crypto_dev char(10) const;           
  hash_value char(1) options(*varsize);
  error_ds likeds(qusec);
end-pr;
```

There are different ways to supply the input data that should be hashed – the most simple way is, to supply a (char)pointer to the data to be hashed, the length of this data chunk and the value `DATA0100` as the the format of the input data.

The algorithm description data structure is also not so complicated – as always simply use likeds(QC3D0100) to declare your own qualified data structure:

```rpgle
dcl-ds QC3D0100 qualified;
  qc3act char(8);
  qc3fof char(1);
end-pr;
```

The first subfield qc3act holds the algorithm context token, that we receive from our call to Qc3CreateAlgorithmContext.

The second subfield qc3fof is the „final operation flag“. A value of ‚0‘ means, that there are more API calls with more data are coming to generate the hash. A value of ‚1‘ means, that the API should finally generate the hash. If you have to hash only one block of data, you cann just call the API once, with qc3fof set to ‚1‘.

### cvthc „API“

The last API we will use is the cvthc API. This is no „normal“ API – even the very short name doesn’t sound like one. cvthc is a machine interface (MI) instruction, but it can be called just like a normal API.

The notation of MI instructions is sometimes a bit strange, because cvthc stands for „convert to hex from char“.

```rpgle
dcl-pr bin2hexAPI extproc('cvthc');
  hexResult       char(65534) options(*varsize);
  binaryInput     char(32767) const options(*varsize);
  inputNibbles    int(10:0) value;
end-pr;
dcl-s hashBin char(64) inz;
dcl-s hashHex char(128) inz;
```

The cvthc instruction simply converts a binary block of data to a hex representation – so it converts a byte with a value of 0xF1 (decimal 241, binary 11110001 or an EBCDIC char ‚1‘) into a string with a value of 'F1'.

This means, that a block of 512 bits (thats 64 bytes) is converted to a string of 128 chars.

I chose the char representation of the hash, because I store it in a database, and this is easier with a string, than with a binary field.

### The main program

The main program looks very simple in fact:

```rpgle
**free
ctl-opt dftactgrp(*no) actgrp(*new);
 
dcl-s hashStr1 like(GetSourceHash) inz;
dcl-s hashStr2 like(GetSourceHash) inz;
 
hashStr1 = GetSourceHash('/QSYS.LIB/QPGMR.LIB/QRPGLESRC.FILE/SHA512.MBR');
hashStr2 = GetSourceHash('/home/QPGMR/sha512.sqlrpgle');

*inlr = *on;
```

I don’t think, that I have to walk thru that code.

### The GetSourceHash procedure

We have our procedure (or function) which takes a varchar(1000) string with the IFS path to the source file or member as a parameter, and returns a varchar(128) string with the hash value in hex notation.

We need some system API includes – the qc2cci member contains the declarations for the crypto APIs, and the qusec member contains the usual error/response data structure for nearly all system APIs.

I also declare some constants and some variables – pretty straight forward.

The first API we need is Qc3CreateAlgorithmContext – it creates a new „context“ to run one of the crypto algorithms. Which algorithm should be used, is configured in the ALGD0500 format.

The next API is Qc3CalculateHash – this one really calculates the hash value of a given set of data. It can be used with different configurations – single shot (e.g. for password hashing) or like we will be using it, in consecutive calls to input data (like the lines of source file) and returning the hash value at the end.

With the parameters crypto_prov and crypto_dev you could even select a dedicated crypto processor, which is installed in your machine – we don’t have one, so we tell the OS to select any crypto solution it likes – if no dedicated hardware is available, this means software.

And of course we need some variables – one holds the binary hash value of 512 bits – this means 64 bytes. The other holds that hash value translated into „readable“ hex notation – this means, that a one-byte value of 0xF1 is translated to a string with two chars „F1“. It also means, that we need double the storage, but strings like there are easier to store.

```rpgle
// Open the file or member
exec sql whenever not found continue;
exec sql whenever sqlwarning continue;
exec sql whenever sqlerror continue;

// read thru IFS file or source member
exec sql declare c1 cursor for
         select line
         from table (qsys2.ifs_read(path_name => :pIfsName,
                                    end_of_line => 'ANY',
                                    maximum_line_length => :#MAX_LINE_LEN,
                                    ignore_errors => 'NO'));
exec sql open c1;
exec sql fetch c1 into :lineDta;
if sqlcode <> *zero and sqlcode <> 100;
  return '';
endif;
```

We have to tell the SQL processor how we want to handle warnings and errors – I always like to handle them myself, so „continue“ is my favorite,

And now that we declared everything, we start to read through the IFS file with the SQL table function ifs_read – and the best, this function also allows to read source members, simply by using the IFS notation of the member, like in the main procedure. No extra work for us.

```rpgle
dsAlgd0500.qc3ha = #SHA512;
initCryptoAPI(dsAlgd0500:'ALGD0500':dsAlgd0100.qc3act:qusec);
dsAlgd0100.qc3fof = '0';
```

We have opened the file or member, now initialize the crypto APIs and get a crypto context, which we need for the the next API. And we also select, that we want to make consecutive calls to the next API.

```rpgle
dow sqlcode = *zero;
  // check the line length
  lineLen = %checkr(' ':lineDta);
  if lineLen = *zero;
    // allways feed the API - empty lines as 1 blank
    lineDta = ' ';
    lineLen = 1;
  endif;
  // call the API for each line
  hashAPI(lineDta:lineLen:'DATA0100':dsAlgd0100:'ALGD0100':'0':'':hashBin:qusec);

  // read next line
  exec sql fetch c1 into :lineDta;
enddo;
exec sql close c1;
```

We loop through the lines of the file or member – ifs_read does remove CR and/or LF chars at the end of the line, so we really only get the data of each line. And it also translates the data according to the CCSID.

In an IFS file, all lines have different line lengths, because trailing blanks are normally removed. This is not the case with source members. So to compare only the „real source code“ without any trailing blanks, we like to remove them – so that the source data from an IFS file, is the same as the source data from a source member.

But it means, that an empty source code line will result in a data chuck with a length of zero – and this cannot be processed by the API, so any completely empty line, will be translated to one space char.

If you step through the program with debug, you will notice, that the „hashBin“ parameter of the API won’t receive a value during the calls inside the loop. This is because, we do the consecutive calls and receive the hash value later.

```rpgle
// finally get the hash value (in binary form) ...
dsAlgd0100.qc3fof = '1';
hashAPI('':0:'DATA0100':dsAlgd0100:'ALGD0100':'0':'':hashBin:qusec);
 
// ... and convert it to a hex string
bin2hexAPI(hashHex:hashBin:%size(hashHex));
 
// return the hex string
return %trim(hashHex);
end-proc;
```

We made it – all source lines were fetched – now we receive the hash value. To do that, we have to set the qc3fof field of the ALGD0100 data structure to „1“ and call the API without data.

In the last step, we use our MI instruction to convert the 512-bit (64 byte) binary hash value to a hex string, and finally we return it to the caller.

I think, this is a good example how to use IBM i system APIs and MI instructions, and I hope you enjoyed the post.
