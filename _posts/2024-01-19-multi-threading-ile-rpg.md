## Multi threading with ILE-RPG

The POSIX API for multithreading - called [`pthreads`](https://man7.org/linux/man-pages/man7/pthreads.7.html) - is the classic way, to run multiple procedures parallel at the same time and in the same process. Using it in ILE-RPG is quite easy, if you know how.

This example also shows, how to use standard C library functions like [`printf()`](https://man7.org/linux/man-pages/man3/printf.3.html) and [`sprintf()`](https://man7.org/linux/man-pages/man3/sprintf.3p.html) in your RPG programs. Especially "sprintf" can be a real time saver to format numbers to your needs, when [`%editc()`](https://www.ibm.com/docs/en/i/7.5?topic=functions-editc-edit-value-using-editcode) or [`%char()`](https://www.ibm.com/docs/en/i/7.5?topic=functions-char-convert-character-data) aren't enough.

Lets break down the program in pieces - we start at the top:

```rpgle
**free
ctl-opt main(main) dftactgrp(*no) actgrp('ATHREAD');
ctl-opt datedit(*dmy/) option(*srcstmt:*nodebugio);

/copy qsysinc/qrpglesrc,pthread        // PThread prototypes
/copy qsysinc/qrpglesrc,unistd         // misc prototypes (e.g. sleep)

// template for worker parameters
dcl-ds worker_parm_t qualified template;
  num int(10:0);
  delay int(10:0);
end-ds;
```

We use free-format RPG - of course - and some standard headers. In this example, I'm using a "linear" main procedure - no RPG cycle, no `*INLR` - and a named activation group. The APIs should work the same with a cycle main procedure or an activation group `*NEW` or even with `QILE` - but I always recommend not to use the QILE activation group.

We don't need special service programs or binder directories to use the pthreads APIs - but we include the system headers "pthread" (maybe obvious) and "unistd" (for the [`sleep()`](https://man7.org/linux/man-pages/man3/sleep.3.html) procedure) from `QSYSINC/QRPGLESRC`.

We want to give our "worker" threads some parameters - and as you can only pass one (!) parameter (a pointer) via the API to your "workers", we create a template data structure for the "caller" and the "workers".

Now to the "Main" procedure:

```rpgle
dcl-proc Main;
  dcl-ds workers dim(4) qualified inz;
    thread likeds(pthread_t);
    parm likeds(worker_parm_t);
  end-ds;
```

Our main procedure is called "Main" (very creative, isn't it?) and has no parameters.

To remember our threads, we use a data structure array `workers`, which consists of two sub data structures - one of type "pthread_t", which is 
defined in the "pthread" include and which will hold the information about each thread.

The other sub data structure uses the worker parameter template, that we defined above. We will store the parameters for each thread there. You will ask yourself, why do we use a seperate parameter structure for each thread?

The answer is simple - as we pass a pointer to that parameter structure, we shouldn't modify it after the thread is "launched". As we want each thread to have its own parameters - isolated from the other threads - we use a separate data structure for each thread.

```rpgle
  dcl-s isAbnormalEnd ind inz;
  dcl-s i int(10:0) inz;
  dcl-s rc int(10:0) inz;
```

We also declare some other variables - a loop counter, the return code of the Pthread APIs and a flag for the on-exit section, to detect whether the procedure was ended normally (by return) or abnormal (by an exception).

```rpgle
  // create 4 threads
  for i = 1 to 4;
    workers(i).parm.num = i;
    workers(i).parm.delay = 6 - i;
    rc = pthread_create(workers(i).thread:*omit:%paddr(worker):%addr(workers(i).parm));
    if rc = 0;
      print('Main: start of worker #'+%char(workers(i).parm.num));
    else;
      print('Main: start of worker #'+%char(workers(i).parm.num)+' failed with RC='+%char(rc));
    endif;
  endfor;

  print('Main: waiting 6 seconds ...-');
  sleep(6);
```

The in next part we use a simple loop to create 4 "worker 
threads. We initialize the worker parameters and call the most important API 
[`pthread_create()`](https://www.ibm.com/docs/en/i/7.5?topic=ssw_ibm_i_75/apis/users_14.html). After that we wait for 6 seconds using the [`sleep()`](https://man7.org/linux/man-pages/man3/sleep.3.html) function.

The print() procedure is just a simple wrapper for the C [`printf()`](https://man7.org/linux/man-pages/man3/printf.3.html) function. As out 
program can only run in batch, the output will land in a spooled file "QPRINT". We will habe a look at the output later. The source code for our `printf()` procedure can be found in the complete source below.

```rpgle
  // cancel 1st worker
  print('Main: cancel worker #1 RC='+%char(pthread_cancel(workers(1).thread)));
  sleep(2);
```

Now something funny - we will cancel the 1st worker thread by calling 
[`phread_cancel()`](https://www.ibm.com/docs/en/i/7.5?topic=ssw_ibm_i_75/apis/users_39.html) passing the matching "pthread_t" data structure. Worker #1 is the longest running thread - so after 6 seconds it is definitely still running.

```rpgle
  print('Main: waiting for all workers ...-');
  for i = 4 downto 1;
    print('Main: joining worker #'+%char(workers(i).parm.num)+' RC='+%char(pthread_join(workers(i).thread:*omit)));
  endfor;

  return;
```

The last part is "joining" the worker thread to the main procedure. This is done by calling [`phread_join()`](https://www.ibm.com/docs/en/i/7.5?topic=ssw_ibm_i_75/apis/users_25.html). Each call waits for the passed thread to finish, detaches it and returns the thread exit status.

```rpgle
  on-exit isAbnormalEnd;
    if isAbnormalEnd;
      print('Main: ending abnormally!');
    else;
      print('Main: ending normally.');
    endif;
end-proc;
```

The last part is the `on-exit` section, this part is executed always - whether the procedure is ended by a regular `return` or by any error, an exception or if the procedure sends an `*escape` message. 

If the end is not due to a regular `return` the `isAbnormalEnd` flag is `*ON`, so you can react to the abnormal ending of your procedure.

Our "Worker" procedure is also quite simple:

```rpgle
  dcl-proc Worker;
  dcl-pi *n;
    parm_ptr pointer value;
  end-pi;

  dcl-ds parm likeds(worker_parm_t) based(parm_ptr);
  dcl-s i int(10:0) inz;
  dcl-s start timestamp(12) inz;
  dcl-s runtime packed(10:5) inz;
  dcl-s isAbnormalEnd ind inz;

  start = %timestamp(*sys:12);
  print('Worker #'+%char(parm.num)+': has thread-id '+getThreadId()+'.');
  print('Worker #'+%char(parm.num)+': setcancelstate RC='+%char(pthread_setcancelstate(PTHREAD_CANCEL_ENABLE:i)));
  print('Worker #'+%char(parm.num)+': setcanceltype RC='+%char(pthread_setcanceltype(PTHREAD_CANCEL_ASYNCHRONOUS:i)));
```

We use a small procedure `getThreadId` which simply retrieves the id of thread and formats it as a string using 
[`sprintf()`](https://man7.org/linux/man-pages/man3/sprintf.3p.html). 
The source code for the `gtThreadId()` procedure can be found in the complete source below.

The worker sets its own thread attributes `cancelstate` and `canceltype`. We want our threads to be "cancelable" and do "asynchronous cancel". This means, that when the thread gets canceled from "outside", it will end immediately.

```rpgle
  for i = 1 to 5;
    print('Worker #'+%char(parm.num)+': waiting '+%char(parm.delay)+' seconds ...-');
    sleep(parm.delay);
  endfor;

  return;
```

The "work" that is done is just for the demonstration and to give the main procedure 
some time.

```rpgle
  on-exit isAbnormalEnd;
    runtime = %diff(%timestamp(*sys:12):start:*seconds:5);
    if isAbnormalEnd;
      print('Worker #'+%char(parm.num)+': ending abnormally after '+%char(runtime)+'s!');
    else;
      print('Worker #'+%char(parm.num)+': ending normally after '+%char(runtime)+'s.');
    endif;
end-proc;
```

There is also the `on-exit` block in the `Worker` procedure. This will not only "catch" runtime exceptions, but also if the thread gets canceled - you will see this here in an example output.

```
...-11.25.49.213: Main: start of worker #1
...-11.25.49.214: Worker #1: has thread-id 00000000:0000002c.
...-11.25.49.215: Worker #1: setcalcelstate RC=0
...-11.25.49.215: Worker #1: setcalceltype RC=0
...-11.25.49.215: Worker #1: waiting 5 seconds ...
...-11.25.49.215: Main: start of worker #2
...-11.25.49.215: Main: start of worker #3
...-11.25.49.215: Main: start of worker #4
...-11.25.49.215: Main: waiting 6 seconds ...
...-11.25.49.215: Worker #2: has thread-id 00000000:0000002d.
...-11.25.49.215: Worker #2: setcalcelstate RC=0
...-11.25.49.215: Worker #2: setcalceltype RC=0
...-11.25.49.215: Worker #2: waiting 4 seconds ...
...-11.25.49.215: Worker #3: has thread-id 00000000:0000002e.
...-11.25.49.215: Worker #3: setcalcelstate RC=0
...-11.25.49.215: Worker #3: setcalceltype RC=0
...-11.25.49.215: Worker #3: waiting 3 seconds ...
...-11.25.49.216: Worker #4: has thread-id 00000000:0000002f.
...-11.25.49.216: Worker #4: setcalcelstate RC=0
...-11.25.49.216: Worker #4: setcalceltype RC=0
...-11.25.49.216: Worker #4: waiting 2 seconds ...
...-11.25.51.246: Worker #4: waiting 2 seconds ...
...-11.25.52.241: Worker #3: waiting 3 seconds ...
...-11.25.53.245: Worker #2: waiting 4 seconds ...           
...-11.25.53.276: Worker #4: waiting 2 seconds ...
...-11.25.54.245: Worker #1: waiting 5 seconds ...
...-11.25.55.245: Worker #3: waiting 3 seconds ...
...-11.25.55.246: Main: cancel worker #1 RC=0
...-11.25.55.246: Worker #1: ending abnormally after 6.03240s
...-11.25.55.306: Worker #4: waiting 2 seconds ...
...-11.25.57.271: Worker #2: waiting 4 seconds ...
...-11.25.57.271: Main: waiting for all workers ...
...-11.25.57.336: Worker #4: waiting 2 seconds ...
...-11.25.58.275: Worker #3: waiting 3 seconds ...
...-11.25.59.366: Worker #4: ending normally after 10.15036s.
...-11.25.59.367: Main: joining worker #4 RC=0
...-11.26.01.301: Worker #2: waiting 4 seconds ...
...-11.26.01.301: Worker #3: waiting 3 seconds ...
...-11.26.04.331: Worker #3: ending normally after 15.11601s.
...-11.26.04.332: Main: joining worker #3 RC=0
...-11.26.05.328: Worker #2: waiting 4 seconds ...
...-11.26.09.358: Worker #2: ending normally after 20.14315s.
...-11.26.09.359: Main: joining worker #2 RC=0
...-11.26.09.359: Main: joining worker #1 RC=0
...-11.26.09.359: Main: ending normally.
```

As you can see, the messages of the `Main` procedure and the workers are interleaved - so the workers are really running in parallel to each other
and to the `Main` - procedure.

The cancelation of worker #1 is taking place immediately - thanks to our thread attributes - and the cancel leads to an abnormal end of the thread - which is nice to know, because you are able, to catch that. 

It is also good to know, that 
[`phread_join()`](https://www.ibm.com/docs/en/i/7.5?topic=ssw_ibm_i_75/apis/users_25.html) is waiting for the given thread to 
end when it is called, and that you won't receive an error, when the thread you are trying to join was already canceled previously.

Last but not least - multi-threading is only allowed for batch jobs which are submitted with the `ALWMLTTHD(*YES)` parameter. Interactive jobs can't use [pthreads](https://man7.org/linux/man-pages/man7/pthreads.7.html) at all.

You can find the complete program source code in my [examples repository](https://github.com/qpgmr-de/examples/blob/main/athread.rpgle) on GitHub. This includes the procedures not shown here and is ready to be cloned or copied and compiled.

I hope you enjoyed this small example.
