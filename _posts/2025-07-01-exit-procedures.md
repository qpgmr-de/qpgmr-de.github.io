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
your program ends normally, you un-register it again, before the program ends. 

But this works only with programs or procedures - what if you want an exit procedure
in the case your activation group ends? 


### Running an exit procedure when an activation group ends

If you want to register an exit procedure from a service program, the other options don't
have the effect, that you need. The `ON-EXIT` works only for a single procedure. And
as a service program doesn't have a "main procedure" you can't register a call stack entry
exit procedure with `CEERTX`.

So an exit procedure for a service program should probably run when the activation group
ends, where the service program is loaded. There are also APIs for this case:

- [`CEE4RAGE`](https://www.ibm.com/docs/api/v1/content/ssw_ibm_i_76/apis/CEE4RAGE.htm)
  Register Activation Group Exit Procedure wirh a 32 bit AG mark
- [`CEE4RAGE2`](https://www.ibm.com/docs/api/v1/content/ssw_ibm_i_76/apis/CEE4RAGE2.htm)
  Register Activation Group Exit Procedure with a 64 bit AG mark
- [`CEE4RAGEL`](https://www.ibm.com/docs/api/v1/content/ssw_ibm_i_76/apis/cee4ragel.htm)
  Register Activation Group Exit Last Procedure (with a 64 bit mark)

The difference between `CEE4RAGE` and `CEE4RAGEL` is simply, that the former is calling 
the exit procedure with a 32 bit activation group marker, wheras the latter is calling
your exit procedure with a 64 bit activation group marker. I think it's safe to say,
that you can use the 64 bit version in every case.

The `CEE4RAGEL` API has the special function, that a exit procedure registered with it
will always be called as the last exit procedure. Whereas the other exit procedures
are called in "LIFO" order - the last registiered procedure gets called first.

Here an example for the calling sequence:

1. `CEE4RAGE2` registering proc1
2. `CEE4RAGE2` registering proc2
3. `CEE4RAGE2` registering proc3
4. `CEE4RAGEL` registering proc4

Now if the activation group ends, the procedures are call in the folling sequence:

1. proc3
2. proc2
3. proc1
4. proc4

Our "proc4" is called last, because it was registered with `CEE4RAGEL` - the other
procedures are called in "LIFO" sequence.

Now to some code is the source code of a small service program. 

```rpgle
**free
ctl-opt nomain;

/include qsysinc/qrpglesrc,qusec

// API prototypes Register Activation Group Exit Procedure
dcl-pr cee4rage2 extproc('CEE4RAGE2');
  *n pointer(*proc) const;
  *n likeds(qusec) options(*omit);
end-pr;
dcl-pr cee4ragel extproc('CEE4RAGEL');
  *n pointer(*proc) const;
  *n likeds(qusec) options(*omit);
end-pr;

// Activation Group Exit procedure reason codes
dcl-c #REASON_CS_ENTRY_CANCELED           x'00000002'; // 0000 0000 0000 0000 0000 0000 0000 0010
dcl-c #REASON_AG_ABNORMAL_END             x'00010000'; // 0000 0000 0000 0001 0000 0000 0000 0000
dcl-c #REASON_AG_ENDING                   x'00020000'; // 0000 0000 0000 0010 0000 0000 0000 0000
dcl-c #REASON_RCLACTGRP                   x'00040000'; // 0000 0000 0000 0100 0000 0000 0000 0000
dcl-c #REASON_JOB_ENDING                  x'00080000'; // 0000 0000 0000 1000 0000 0000 0000 0000
dcl-c #REASON_EXIT_VERB                   x'00100000'; // 0000 0000 0001 0000 0000 0000 0000 0000
dcl-c #REASON_UNHANDLED_FUNCTION_CHECK    x'00200000'; // 0000 0000 0010 0000 0000 0000 0000 0000
dcl-c #REASON_OUT_OF_SCOPE_JUMP           x'00400000'; // 0000 0000 0100 0000 0000 0000 0000 0000

// Activation Group Exit procedure result_codes (passed in and out)
dcl-c #RESULT_CODE_NO_ACTION              0; // Do not change the action
dcl-c #RESULT_CODE_RECOVERED             10; // Recovered from previous result code 20
dcl-c #RESULT_CODE_FAIL_CONTROLLED       20; // Failed but execute remaining exit procs
dcl-c #RESULT_CODE_FAIL_IMMEDIATLY       21; // Fail immediatly, don't execute remaining exit procs

// the procedure registering the exits
dcl-proc someProcedure export;
  dcl-pi *n extproc(*dclcase);
  end-pi;

  cee4rage2(%proc(exitProc1):*omit);
  cee4ragel(%proc(exitProc2):*omit);

  return;
end-proc;

// Activation Group Exit procedure #1
dcl-proc exitProc1 export;
  dcl-pi *n extproc(*dclcase);
    ag_mark uns(20) const;
    reason uns(10) const;
    result_code uns(10);
    user_rc uns(10);
  end-pi;

  if %bitand(reason:#REASON_AG_ABNORMAL_END) = #REASON_AG_ABNORMAL_END;
    snd-msg *info %proc()+': activation group exit ending abnormally!';
    result_code = #RESULT_CODE_FAIL_CONTROLLED;
  else;
    snd-msg *info %proc()+': activation group exit ending normally.';
  endif
  
  return;
end-proc;

// Activation Group Exit procedure #2
dcl-proc exitProc2 export;
  dcl-pi *n extproc(*dclcase);
    ag_mark uns(20) const;
    reason uns(10) const;
    result_code uns(10);
    user_rc uns(10);
  end-pi;

  if result_code = #RESULT_CODE_FAIL_CONTROLLED;
    snd-msg *info %proc()+': activation group exit recovering from error ...';
    result_code = #RESULT_CODE_RECOVERED;
  endif

  snd-msg *info %proc()+': activation group exit ending normally.';
  
  return;
end-proc;
```

The procedure interface of `CEE4RAGE2` and `CEE4RAGEL` is exactly the same - with `CEE4RAGE` the
first parameter would also be of type `uns(10)` as it uses the 32-bit activation group marker.
So as I already wrote, it's quite safe to ignore `CEE4RAGE` and always use `CEE4RAGE2` and
`CEE4RAGEL` instead.




### Finally

As long, as your initialization doesn't need any parameters from the calling program, this
technique is quite useful, and you will never have to think about calling your initializer.

So I hope you found our little *excursion* into C++ and object-orientation entertaning and 
easy to follow.

If you have questions or ideas - I have opened the [Discussions](https://github.com/qpgmr-de/qpgmr-de.github.io/discussions)
page of this repository - if you like, leave a comment.
