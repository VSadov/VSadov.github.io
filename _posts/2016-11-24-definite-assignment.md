---
title: Definite Assignment Analysis of locals. The real purpose.
tags: [CSharp, Definite Assignment, Language Design]
---

Definite Assignment Analysis prevents bugs, but there are deeper reasons to have it as a required feature in the language.

Indeed - Why have such strict rules instead of just assuming that unassigned locals contain default/zero values? After all, that seems to work ok for fields. It is also known that C# compiler decorates all methods with IL directive ```localsinit```, which guarantees that all locals are zeroed out when method is entered. So what is the problem?

```localsinit``` is, unfortunately, not enough to implement the semantics of C# locals. It would work if the life time of all local bindings (i.e. [extents](https://en.wikipedia.org/wiki/Variable_(computer_science)#Scope_and_extent)), was the whole method, but that is not the case in C#. In C# locals can have scopes smaller than the entirety of a method and extents match the lexical scopes. Every time the control flow enters a scope, a new set of bindings for the locals contained by that scope is supposed to be created and the bindings exist as long as they can be referenced. In the most general sense the "new bindings" would imply a newly allocated storage completely unrelated to the bindings possibly created when the same scope was entered previously.

A brute-force solution would be to map local variables of the same scope to fields in a synthesized class and create a new instance of such class when entering a scope. In fact this is what happens to locals that are accessible from lambda expressions. Such locals can be used beyond the life time of the containing method and multiple bindings to the same variables could coexist at the same time, so compiler needs to allocate their storage on the heap and rely on GC for keeping them alive as long as they can be referenced.

Example of multiple bindings to the same local:  

```cs
    class Program
    {
        static void Main(string[] args)
        {
            int iteration = 0;

            var setters = new Action<int>[2];
            var getters = new Func<int>[2];

          reenterScope:

            {
                int sameVariable = 0;  // <-- THE VARIABLE

                setters[iteration] = (i) => sameVariable = i;
                getters[iteration] = () => sameVariable;
            }

            if (iteration++ < 1)
            {
                goto reenterScope;
            }

            Console.WriteLine("Original values of different bindings of the sameVariable: ");

            Console.WriteLine(getters[0]());
            Console.WriteLine(getters[1]());
            Console.WriteLine();

            setters[0](33);
            setters[1](42);

            Console.WriteLine("Assigned values of different bindings of the sameVariable: ");

            Console.WriteLine(getters[0]());
            Console.WriteLine(getters[1]());
        }
    }
```  
```
Original values of different bindings of the sameVariable:
0
0

Assigned values of different bindings of the sameVariable:
33
42
```

The most common case is, however, when locals are just that - locals. They are not accessed from lambdas or anything like that and at any time only one (or none) bindings to such local may exist. In such cases locals can be simply mapped to IL local slots and reused every time the control flow enters the scope.
The only problem is that the slot values would need to be “reset” every time the scope is entered to the default value and there is no help from ```localsinit``` here since that works only once - when the whole method is invoked.

In theory, compiler could inject code that would do the “resetting” of all relevant slots, when a scope is entered, but that would be wasteful. Only some of the locals in a given scope would be read from. Besides, most of them would be written to before reading anyways, so why not just require that a local is written to before being read? That would make the code less buggy, but most of all it will make the “resetting” entirely unnecessary.

**Essentially, a rule that requires that locals are definitely assigned before being read serves the same purpose as ```localsinit```, but does much better job.**

1. It works at every nested lexical scope recursively (not just on the method level)  
2. It gives stronger guarantees. You can see only what you have already assigned to the variable. It is impossible to read uninitialized/stale state by accident.  
3. It is minimally redundant. If you do not read a local on some code path you do not need to ensure that it is assigned on that code path  

Simple example of some variables assigned on one code path and not assigned on the other. As long as we do not read the variable it is ok to have it not assigned.

```cs
static void Main(string[] args)
{
    int a;
    int b;

    if (args.Length > 0)
    {
        goto path1;
    }
    else
    {
        goto path2;
    }

    path1:
    // assign only "a"
    a = 123;
    goto path1continues;

    path2:
    // assign only "b"
    b = 345;
    goto path2continues;

    path1continues:
    // on this codepath "b" is not assignd,
    // but that is ok since we do not read it.
    Console.WriteLine(a);
    goto exit;

    path2continues:
    // on this codepath "a" is not assignd,
    // but that is ok since we do not read it.
    Console.WriteLine(b);

    exit:
    return;
}
```


Interestingly, in VB, for historical reasons, locals not referenced from lambdas do have extents that match the entirety of the method and thus definite assignment analysis is much less strict - it basically exists just to give warnings on some cases that likely to be coding mistakes.

example of a VB local binding maintained through the entirety of the method life time:

```vb  
Module Module1

    Sub Main()

        Dim iterations As Integer = 3

        While iterations > 0

            Console.WriteLine("variable declared and initialized")
            Dim variable As Integer = 42

reentryTheScope:
            Console.WriteLine(variable)

            variable += 1
            iterations -= 1
        End While


        If iterations > -3 Then
            ' "variable" is out of scope here, but it exists and has a value
            ' let's reenter the While loop and check upon the value of the "variable"
            Console.WriteLine("reentering scope")
            GoTo reentryTheScope
        End If

    End Sub

End Module
```  
```
variable declared and initialized
42
variable declared and initialized
42
variable declared and initialized
42
reentering scope
43
reentering scope
44
reentering scope
45
```

The locals captured into lambda closures, however, have scoped extents in VB - surely the lifetimes cannot be bound to the lifetime of the containing method anymore when lambdas are involved. Similarly to C#, fresh bindings for captured locals are created when scope is entered and their lifetimes are bound to the lifetime of the referencing lambdas. So if locals are captured, the example above would start behaving differently. To make the unfortunate inconsistency less observable, VB refuses to compile code like above when locals are captured.

```vb  
Module Module1

    Sub Main()

        Dim iterations As Integer = 3

        While iterations > 0

            Console.WriteLine("variable declared and initialized")
            Dim variable As Integer = 42

reentryTheScope:
            Console.WriteLine(variable)

            variable += 1
            iterations -= 1

            ' cause "variable" to be captured into a closure
            Dim lambda As Func(Of Integer) = Function() variable
        End While


        If iterations > -3 Then
            ' "variable" is out of scope here, but it exists and has a value
            ' let's reentering the While loop and check upon the value of the "variable"
            Console.WriteLine("reentering scope")
            GoTo reentryTheScope
        End If

    End Sub

End Module
```  
```
Module1.vb(27, 13) : Error BC36597 : 'Goto reentryTheScope' is not valid because 'reentryTheScope' is inside a scope that defines a variable that is used in a lambda or query expression.
```


**Pedantic observations:**

Since C# enforces stronger invariant than provided by ```localsinit```, one would wonder why compiler still puts ```localsinit``` on methods. A simple answer is that IL verification rules require that. The underlying reason for the requirement is that the user’s code is not the only entity that might read the locals. The other one is the Garbage Collector.  

The issue with GC is that it scans the IL locals of currently active methods in order to record the roots of reachable object graphs, and GC happens at fairly random times. Definite assignment does not guarantee that locals will be assigned something deterministic before GC happens and things will go terribly bad if locals contain random junk. Therefore there is a rule that requires that verifiable methods have ```localsinit``` as an instruction directing the JIT to add a method preamble that wipes the whole stack frame clean before the method body is formally entered and GC had any chance to scan the locals.  

In theory the rule could be required only on methods with locals of reference types  (or structs containing references), but that would make a difference only to a fraction of methods while complicating the rule. Instead CLI standard allows JIT implementations to disregard the directive if, through some analysis, it could be inferred that not wiping the frame is a safe thing to do.  

I am not sure if JITs use this kink in the rules very often though. With exception of the most trivial cases, the analysis could be too involved to be feasible at JIT time and wiping the stack frame is not overly expensive. Still, since there are some costs associated with locals (wiping the frame is just one of them), C# compiler generally tries to be frugal with usage of local slots, especially when compiling with /o+.  
