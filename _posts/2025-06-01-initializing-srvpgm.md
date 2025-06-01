## Automatically initializing ILE-RPG service programs

Sometimes your service program needs some initialization before it is able to run. 
But calling an initialization procedure in each of the programs, that uses the 
service program, is somehow inconvenient.

Sadly, the is no mechanism in ILE-RPG that helps with automatically running a procedure
when the service program is loaded. But lately I stumbled over a ["gem" in IBM i Info Center](https://www.ibm.com/support/pages/initializing-context-during-service-program-activation)
that describes a way, to create exactly that - and the hero in this tale is: C++

### The problem

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
[`CEE4RAGE2`](https://www.ibm.com/docs/api/v1/content/ssw_ibm_i_76/apis/CEE4RAGE2.htm) API.

Now you would have to call `initSrvpgm()` in each program that uses your service program. And 
you would have to call it exactly **once** - because in our example, you only want to register the
activation group exit procedure once, as it only should run exactly once, when the activation
group ends.

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

You can even hide the `initSrvpgm` procedure from being called by someone else, by leaving the
export out of the binder language source. But since it has to be called by the C++ module inside
the service program, it still has to be flagged as `export` in the RPG source code.

The [page in IBM i Info Center](https://www.ibm.com/support/pages/initializing-context-during-service-program-activation)
is recommending that you only create the module only once, and always use the same module and
the same procedure name for the init procedure in RPG. But I prefer to have a separate module
for each service program, as I also like to prefix the procedure names of my service programs.

At this point you are free to stop reading - in fact, up until now, I haven't written anything,
that you cannot read on the [page in IBM i Info Center](https://www.ibm.com/support/pages/initializing-context-during-service-program-activation).
But in the next chapter, I will at least try to explain, why or how this mechanism works.

### The explanation

No let`s go through the C++ code, to find out, what happens.

```cpp
extern "C" void initSrvpgm(void);
```

This is simply a C/C++ function prototype for our ILE-RPG procedure. As as C++ is a 
case-sensitive language, and I have used the keyword `extproc(*dclcase)` with our procedure, 
the case has to match exactly the external procedure name or export. 

Without the `extproc(*dclcase)` keyword, the prototyp would have to be `INITSRVPGM`, as ILE-RPG
is normally exporting every procedure in UPPERCASE. But I like camelCase - not only in my 
source code - also as the exported procedure name.

```cpp
class InitSrvpgmClass {                    
  public:                              
    InitSrvpgmClass() {      
      initSrvpgm();  
    };
};                                    
```

In this part, a C++ class is defined. Without going into too much details, C++ is an 
object-oriented language, and a class is some kind of template that can mix data (like a 
data structure) and code (like procedures, but called methods).

And the part with `InitSrvpgmClass() { ... }` is a class constructor method, that automatically
gets called, when a new instance of our class - meaning a new variable of that type - is created.
A constructor method must always have the same name - including matching case - as the class
itself - because your know ... C++ is a case-sensitive language.

```cpp
InitSrvpgmClass dummy;
```

That last part is a module global variable definition. In C++ you can declare variables
outside of your functions - and exaclty like ILE-RPG those variables are global to the 
module.

But the *trick* is, that the variable contains code. With the definition of the variable,
the class constructor is automatically called - and the constructor calls our `initSrvpgm()`.

So the *magic* is simply an object-oriented feature of C++, that we are *abusing* for
our advantage.

### Finally

As long, as your initialization doesn't need any parameters from the calling program, this
technique is quite useful, and you will never have to think about calling your initializer.

I hope you found our little *excursion* into C++ and object-orientation entertaning and 
easy to follow.

If you want to discuss, leave a comment or send me a message, please open an 
[issue](https://github.com/qpgmr-de/qpgmr-de.github.io/issues).

