## Running "automatic" exit procedures

In programming, cleaning up behind you is important. Nobody wants to leave a mess when 
a procedure, program, service program or job ends - even if it ends "unplaned" (with 
an error or exception), "furcefully" (like `ENDJOB`) or perfectly fine. 

In this and upcoming posts, I want to discuss and descript the common options that we have, 
to gracefully end our procedures, programs, service programs or jobs, even when an error or 
exception occurs or the job is ended unplaned.

First we take a look not only at the most modern option, but also at easiest to implement - 
the `ON-EXIT` statement in ILE-RPG.

### Running an exit routine after your procedure with `ON-EXIT`

In [2016 ILE-RPG was extended](http://ibm.biz/RPG_ON_EXIT_Section) with the 
[`ON-EXIT`](https://www.ibm.com/docs/en/i/7.6.0?topic=codes-exit-exit) keyword/statement. 
It creates a "sub-procedure" inside your own procedure - and this "sub-procedure" is 
executed whenever(!) your procedure ends - whether it's a normal end (by using return) 
or an abnormal end (by an exception).

Even when the job, in which your procedure runs, is ended with `*IMMED` the `ON-EXIT` 
section will still be executed.

Here a small example to see how easy this is:
```rpgle
**free
ctl-opt actgrp(*new);

myProcedure();
*inlr = *on;
retrurn;

dcl-proc myProcedure;
  dcl-pi *n ind extproc(*dclcase);
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
i.e. due to an exception - the flag `isAbnormalEnd` in set to `*ON`. You have to declare the 
`isAbnormalEnd` indicator (boolean) variable - it can be procedure local or globally declared.

What happens is, that the `ON-EXIT` sub-procedure is executed, whether a `RETURN` is executed or an
unhandled exception occurs. Exceptions that are handled by a `MONITOR` group or by an `E` operation
extender will **NOT** trigger the `ON-EXIT` procedure.

Please try it - compile the program and call it. In the job log you should see 2 messages:

```
  This is the myProcedure procedure ...                         
  This is the ON-EXIT procedure called _QRNI_ON_EXIT_myProcedure
```

As you can see - after the `RETURN`, the `ON-EXIT` sub-procedure was executed - and `isAbnormalEnd`
was **NOT** active.

### `ON-EXIT` when an exception occurs

Now we want to test the abnormal end - so we modify like this:
```rpgle
dcl-proc myProcedure;
  dcl-pi *n ind extproc(*dclcase);
  end-pi;

  dcl-s isAbnormalEnd ind inz;
  dcl-s num int(10) inz;

  snd-msg *info 'This is the '+%proc()+' procedure ...';

  num = num / num; // -> exception will occur

  snd-msg *info 'Just before RETURN from '+%proc()+' ...';
  return *on;

  on-exit isAbnormalEnd;
    snd-msg *info 'This is the ON-EXIT procedure called '+%proc(*onexit);
    if isAbnormalEnd;
      snd-msg *info 'Procedure '+%proc(*owner)+' ended abnormal!';
      return *off;
    endif;
end-proc;
```

If we compile and call the program now, you will receive an exception message, and the 
follwowing messages in your job log:

```
  This is the myProcedure procedure ...                          
  Attempt made to divide by zero for fixed point operation.      
  The call to myProcedur ended in error (C G D F).               
? C                                                              
  This is the ON-EXIT procedure called _QRNI_ON_EXIT_myProcedure 
  Procedure myProcedure ended abnormal!                          
  Application error.  MCH1211 unmonitored by ON_EXIT at statement 0000000017, instruction X'0000'.                             
```

Nice - even as our procedure never reached the `RETURN` statement, the `ON-EXIT` sub-procedure
is still executed. But this time, `isAbnormalEnd` is set to `*ON`.

### `ON-EXIT` when the job or program is ended *forcefully*

What happens, when we end the program or job *forcefully*. Like with `SysReq` option 2 or with
an `ENDJOB` command. So let's modify our procedure again - this time with an endless loop:

```rpgle
dcl-proc dcl-proc myProcedure;
  dcl-pi *n ind extproc(*dclcase);
  end-pi;

  dcl-s isAbnormalEnd ind inz;
  dcl-pr sleep uns(10) extproc('sleep');
    seconds uns(10) value;
  end-pr;

  snd-msg *info 'This is the '+%proc()+' procedure ...';

  dow *on;
    snd-msg *info 'Inside the endless loop ...';
    sleep(5); // sleeping for 5 seconds
  enddo;

  snd-msg *info 'We should never reach this ...';
  return *on;

  on-exit isAbnormalEnd;
    snd-msg *info 'This is the ON-EXIT procedure called '+%proc(*onexit);
    if isAbnormalEnd;
      snd-msg *info 'Procedure '+%proc(*owner)+' ended abnormal!';
      return *off;
    endif;
end-proc;
```

We compile the program and call it form the command line - it will run endlessly, so
we interrupt it with `SysReq 2` - and we find this in the job log:
```
  This is the myProcedure procedure ...                         
  Inside the endless loop ...                                   
  Inside the endless loop ...                                   
>  *SYSTEM/ENDRQS                                               
  This is the ON-EXIT procedure called _QRNI_ON_EXIT_myProcedure
  Procedure myProcedure ended abnormal!                         
```

Like in the case of the exception, the `ON-EXIT` gets executed, and `isAbnormalEnd` is
`*ON`. 

Now submit the program into the background with `SBMJOB CMD(CALL PGM(ON_EXIT)) JOB(ON_EXIT)` 
and check for the running job with `WRKSBMJOB`. Find your job, and use option `4` and `F4`. 
You can end the job with `OPTION(*CNTRLD)` or with `OPTION(*IMMED)` - it doesn't matter.

If we inspect the printed job log, we will also find the messages from the `ON-EXIT`
sub-procedure - even when we used `OPTION(*IMMED)`. 

This means, that `*IMMED` is not so *instantly* as we always thought. Our `ON-EXIT`
sub-procedure, and also all other `ON-EXIT` sub-procedures in the call-stack will be
executed, even when we try to end the job as fast as we can.

### What else can `ON-EXIT` can do?

Inside the `ON-EXIT` sub-procedure you have several options. You get the name of LR
*original* procedure with `%PROC(*OWNER)`. Whereas `%PROC(*ONEXIT)` is returning the 
name of the `ON-EXIT` "sub-procedure" - in our case this is `_QRNI_ON_EXIT_myProcedure`.

An interesting option is, that from inside the `ON-EXIT` sub-procedure, you can also return 
a completly other return value, than that of the original procedure. If you don't code a 
`RETURN` in `ON-EXIT` or if that `RETURN` is conditioned (like in our example) the original
return value is returned.

Inside the `ON-EXIT` sub-procedure you have access to all variables defined inside the
owner procedure. You can also access all global variables. This means, the `ON-EXIT` 
sub-procedure runs in the same scope as the owner procedure.

### When can we use it?

The `ON-EXIT` opcode is available for every procedure. In the case of a `main` procedure,
it's only available in **linear** `main` procedures - meaning those, that are indicated by
the control keyword `MAIN(...)`.

You cannot use `ON-EXIT` inside a **cycle** main procedure - those where you have to set 
`*ILR` to `*ON` to end the program (like in our example). But you can use them in each and 
every procedure that might get called from than cycle main procedure.

### `ON-EXIT`

The `ON-EXIT` sub-procedure is by far the easiest and most straight-forward solution, to 
clean up. I use it all the time - closing files or SQL cursors, checking for correct 
execution, logging errors and addition information and much more. 

I hope you enyoyed this post
