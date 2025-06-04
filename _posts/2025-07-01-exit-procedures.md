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
[`CEE4RAGEL`](https://www.ibm.com/docs/api/v1/content/ssw_ibm_i_76/apis/CEE4RAGEL.htm) APIL

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

Let's assume the `initSrvpgm` procedure is initializing some global variables, is setting up
something in QTEMP or is registering an activation group exit procedure with the 


Now you would have to call `initSrvpgm()` in each program that uses your service program. And 
you would have to call it exactly **once** - because in our example, you only want to register 
the activation group exit procedure once, as it only should run exactly once, when the 
activation group ends.

This is rather inconvenient and also error prone. If the developer forgets about it, the
initialization isn't done - or maybe worse, it's done twice.

### The solution

We need some code, that is automatically executed, when the service program is loaded and 
activated. But sadly neither ILE-RPG, ILE-COBOL, ILE-CL nor ILE-C have a mechanism or construct
to do that.

But there is another ILE family member that can do exactly this: **ILE-C++**

```cpp
extern "C" void initSrvpgm(void);

class InitSrvpgmClass {                    
  public:                              
    InitSrvpgmClass() {      
      initSrvpgm();  
    };
};                                    

InitSrvpgmClass dummy;
```

This code should be in a member of type `CPP` (or an IFS source file with `.cpp` extension).

You would compile it with the `CRTCPPMOD` command to a module. And then bind both the ILE-RPG
and the ILE-C++ module with `CRTSRVPGM` to a service program. And if a program that is linked 
to the service program is loaded, the service program also gets loaded - and like 
*auto-magically* the `initSrvpgm` procedure gets called.

You can even hide the ILE-RPG procedure `initSrvpgm` from being called by someone else, by 
leaving the export out of the binder language source. But since it has to be called by the 
C++ module inside the service program, it still has to be flagged as `export` in the RPG 
source code, to work.

The [page in IBM i Info Center](https://www.ibm.com/support/pages/initializing-context-during-service-program-activation)
is recommending that you create the module only once, and always use the same module and
the same procedure name for the init procedure in RPG. But I prefer to have a separate module
for each service program, as I also like to prefix the procedure names of my service programs.

At this point you are free to stop reading - in fact, up until now, I haven't written anything,
that you cannot read on the 
[page in IBM i Info Center](https://www.ibm.com/support/pages/initializing-context-during-service-program-activation). 
But in the next chapter, I will at least try to explain, why or how this mechanism works.

### The explanation

Now let`s go through the C++ code, to find out, what exactly happens.

The first line reads as:

```cpp
extern "C" void initSrvpgm(void);
```

This is simply a C/C++ function prototype for our ILE-RPG procedure. As C++ is a 
case-sensitive language, and I have used the keyword `extproc(*dclcase)` with our procedure, 
the case has to match exactly the external procedure name or export. 

Without the `extproc(*dclcase)` keyword, the prototyp would have to be `INITSRVPGM`, as ILE-RPG
is normally exporting every procedure in UPPERCASE. But I like camelCase - not only in my 
source code - also as the exported procedure name.

The next part is:

```cpp
class InitSrvpgmClass {                    
  public:                              
    InitSrvpgmClass() {      
      initSrvpgm();  
    };
};                                    
```

Here, we define a C++ class. Without going into too much details, C++ is an 
object-oriented language, and a class is some kind of template that can mix data (like a 
data structure) and code (like procedures, but called methods). Those classes ("templates")
are later used to create an object (a "variable") of that class ("type").

The part with `InitSrvpgmClass() { ... }` is a class constructor method, that automatically
gets called, when a new instance of our class - meaning a new variable of that type - is created.
A constructor method must always have the same name - including matching case - as the class
itself - because your know ... C++ is a case-sensitive language.

The last part is: 

```cpp
InitSrvpgmClass dummy;
```

This is simply a module global variable definition. In C++ you can declare variables
outside of your functions - and exaclty like ILE-RPG those variables are global to the 
module. And those variables are "initialized" at the moment, the module is loaded.

And here the *trick* - as we defined a class constructor, that constructor code is executed 
in the very moment, where the variable is initialized - *... when the module is loaded*. 
And because our constructor is calling our ILE-RPG procedure `initSrvpgm()`, our ILE-RPG code 
is also executed - *... when the (C++) module is loaded*.

So that *Kind of Magic* is simply an object-oriented feature of C++, which we are 
*exploiting shamelessly* for our own advantage.

### Finally

As long, as your initialization doesn't need any parameters from the calling program, this
technique is quite useful, and you will never have to think about calling your initializer.

So I hope you found our little *excursion* into C++ and object-orientation entertaning and 
easy to follow.

If you have questions or ideas - I have opened the [Discussions](https://github.com/qpgmr-de/qpgmr-de.github.io/discussions)
page of this repository - if you like, leave a comment.
