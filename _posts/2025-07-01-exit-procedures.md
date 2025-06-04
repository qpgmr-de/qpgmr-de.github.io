## Running "automatic" exit-procedures

Sometimes you want to run a procedure to clean up behind yourself. 
Whether its a program or procedure - or maybe a service program.

I want to discuss the available options with you - every option has 

### Running an exit routine after your procedure

Since some time, ILE-RPG has the statement [`ON-EXIT`](https://www.ibm.com/docs/en/i/7.6.0?topic=codes-exit-exit). 
It creates some sort of "sub-procedure" inside your procedure - and this "sub-procedure" is executed
whenever (!) your procedure ends - whether it's normal (by using return) or abnormal (by an exception).

Here we have a small example:

```rpgle
dcl-proc myProcedure;
  dcl-pi *n ind;
  end-pi;

  dcl-s isAbnormalEnd ind inz;

  snd-msg *info 'This is the '+%proc()+' procedure ...';

  return *on;

  on-exit isAbnormalEnd;
    snd-msg *info 'This is the ON-EXIT procedure called '+%proc(*onexit);
    if isAbnormalEnd;
      snd-msg *info 'Procedure '+%proc(*owner)+' ended abnormal!';
      return *off;
    endif;
end-proc;
```

So whenever this procedure ends, the `ON-EXIT` block is executed. If the procedure is ending abnormal - 
i.e. due to an exception - the flag `isAbnormalEnd` in `*ON`.

Inside the `ON-EXIT` you have several options. You get the name of the "original" procedure with 
`%PROC(*OWNER)`. Whereas `%PROC(*ONEXIT)` is returning the name of the `ON-EXIT` "sub-procedure" - 
in our case that would be `_QRNI_ON_EXIT_myProcedure`.

An interesting option is, that you can return an other value, that the original procedure. If you
don't code a `RETURN` in `ON-EXIT` or if that `RETURN` is conditioned (like in our example) the
`ON-EXIT` "sub-procedure" returns the otiginal return value.

The `ON-EXIT` opcode is only available for "linear" procedures. So `ON-EXIT` is no option, if you
use a "cycle" main procedure. But for this case, you have the next option.


### Running an exit procedure when a call stack entry ends

If you want to run an exit procedure after a program or procedure ends, you can easily register an
exit procedure with the [`CEERTX`](https://www.ibm.com/docs/en/i/7.6.0?topic=ssw_ibm_i_76/apis/CEERTX.html) API.

Here is a small example:

```rpgle
**free

/include qsysinc/qrpglesrc,qusec

// API prototype Register Exit Procedure
dcl-pr ceertx extproc('CEERTX');
  *n pointer(*proc) const;
  *n pointer const options(*omit);
  *n likeds(qusec) options(*omit);
end-pr;

// register exit handler
ceertx(%paddr(myExitHandler):*omit:*omit);

snd-msg *info %proc()+': (cycle) main procedure.';

*inlr = *on;
return;

dcl-proc myExitHandler;
  dcl-pi *n extproc(*dclcase);
  end-pi;

  snd-msg *info %proc()+': exit handler.';

  return;
end-proc;
```

First we have to create a prototype for the 
[`CEERTX`](https://www.ibm.com/docs/en/i/7.6.0?topic=ssw_ibm_i_76/apis/CEERTX.html) API.
Basically the API has 3 parameters. 

The 1st is a procedure pointer to your "exit handler"
procedure - the procedure, that should run, when the program (or exactly the call stack
entry) is ending. The other 2 parameters can be omitted - and discussing them is a
story for another day.

You can also un-register the exit handler, using the 
[`CEEUTX`](https://www.ibm.com/docs/en/i/7.6.0?topic=ssw_ibm_i_76/apis/CEEUTX.html) API.
Here is the prototype:

```rpgle
// API prototype Unregister Exit Procedure
dcl-pr ceeutx extproc('CEEUTX');
  *n pointer(*proc) const;
  *n likeds(qusec) options(*omit);
end-pr;
```

As an example, you register your exit handler for the case an error occurs. And in case
your program ends normally, you un-register it again. 


### Running an exit procedure when an activation group ends

[`CEE4RAGE`](https://www.ibm.com/docs/api/v1/content/ssw_ibm_i_76/apis/CEE4RAGE.htm) API.

[`CEE4RAGE2`](https://www.ibm.com/docs/api/v1/content/ssw_ibm_i_76/apis/CEE4RAGE2.htm) API.

[`CEE4RAGEL`](https://www.ibm.com/docs/api/v1/content/ssw_ibm_i_76/apis/cee4ragel.htm) APIL

Here is the source code of a small service program. 

```rpgle
**free
ctl-opt nomain;

dcl-proc initSrvpgm export;
  dcl-pi *n extproc(*dclcase);
  end-pi;

  snd-msg *info %proc()+': initializing service program ...';
  // do some initialization
  return;
end-proc;

dcl-proc doSomething export;
  dcl-pi *n extproc(*dclcase);
  end-pi;

  snd-msg *info %proc()+': doing something ...';
  // do something 
  return;
end-proc;
```


### Finally

As long, as your initialization doesn't need any parameters from the calling program, this
technique is quite useful, and you will never have to think about calling your initializer.

So I hope you found our little *excursion* into C++ and object-orientation entertaning and 
easy to follow.

If you have questions or ideas - I have opened the [Discussions](https://github.com/qpgmr-de/qpgmr-de.github.io/discussions)
page of this repository - if you like, leave a comment.
