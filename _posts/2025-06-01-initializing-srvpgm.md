## Automatically initializing ILE-RPG service programs

Sometimes your service program needs some initialization before it is able to run. 
But calling an initialization procedure in each of the programs, that uses the 
service program, is somehow inconvenient.

Sadly, there is no mechanism in ILE-RPG that helps with automatically running a 
procedure when a service program is loaded. But lately I stumbled over a 
["gem" in IBM i Info Center](https://www.ibm.com/support/pages/initializing-context-during-service-program-activation)
that describes a way, to create exactly that - and the hero in this tale is: C++

### The problem

Here is the source code of a small service program. 

```rpg
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
[`CEE4RAGE2`](https://www.ibm.com/docs/api/v1/content/ssw_ibm_i_76/apis/CEE4RAGE2.htm) API.

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

