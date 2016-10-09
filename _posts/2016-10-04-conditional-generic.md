---
title: Conditional access operator (generic receivers).
tags: [CSharp, Conditional Operator]
---
Generally conditional access operator is applicable only to variables of reference or nullable types. Essentially the receiver must be able to have a null value. An interesting case arises when the type of the receiver is a generic type parameter.

The restriction for generics is basically "some substitutions of T can be types that can have null value". Note that it is not required for _all_ substitutions to allow nulls. Such restriction was considered and rejected for being too constraining. As a result, with generics, you may have a situation where receiver can be either a struct or a class and conditional access is still permitted.

Consider the following example:

```cs
    class Program
    {
        static void Main(string[] args)
        {
            C1[] c = new C1[] { new C1() };
            S1[] s = new S1[] { new S1() };

            // Should print "True"
            SafeDispose(ref c[0]);
            System.Console.WriteLine(c[0].disposed);

            // Should print "True"
            SafeDispose(ref s[0]);
            System.Console.WriteLine(s[0].disposed);
        }

        static void SafeDispose<T>(ref T d) where T: IDisposable
        {
            d?.Dispose();
        }

        class C1 : IDisposable
        {
            public bool disposed;

            public void Dispose()
            {
                disposed = true;
            }
        }

        struct S1 : IDisposable
        {
            public bool disposed;

            public void Dispose()
            {
                disposed = true;
            }
        }
    }

```

Prints:

```
True
True
```

Possible naive implementation of SafeDispose :

```cs
static void SafeDispose<T>(ref T d) where T: IDisposable
{
    // d?.Dispose();

    T temp = d;         //capture the field into a temporary local
    if (temp != null)   //test the temporary
    {
        temp.Dispose(); //invoke on a variable that is certainly not null
    }
}
```

Prints:

```
True
False
```

The problem here is that when T is a class, we need to capture the receiver in a temp (see explanation in an [earlier post]({% post_url 2016-10-01-conditional-receiver %})). That, however, does not work when T happens to be a struct. When we invoke the member on a copy, all the mutations done by the member would be applied to the copy and lost. It is not common for structs to implement interfaces that imply mutations, but it is possible and compiler needs to handle such cases correctly.
In fact we can have a method instantiated with a class and a struct in the same program and both cases must work.

Compiler handles this case by making the receiver cloning code conditional on additional check - whether T happens to be a type that can be ```null``` at all. If given T cannot be ```null```, the null check is unnecessary and thus cloning is unnecessary too.

The actual IL emitted by the compiler for ```d?.Dispose()``` looks like the following -

```cs
.method private hidebysig static
	void SafeDispose<([mscorlib]System.IDisposable) T> (
		!!T& d
	) cil managed
{
	// Method begins at RVA 0x20a8
	// Code size 47 (0x2f)
	.maxstack 2
	.locals init (
		[0] !!T
	)

	IL_0000: ldarg.0
	IL_0001: ldloca.s 0
	IL_0003: initobj !!T
	IL_0009: ldloc.0
	IL_000a: box !!T
	IL_000f: brtrue.s IL_0023

	IL_0011: ldobj !!T
	IL_0016: stloc.0
	IL_0017: ldloca.s 0
	IL_0019: ldloc.0
	IL_001a: box !!T
	IL_001f: brtrue.s IL_0023

	IL_0021: pop
	IL_0022: ret

	IL_0023: constrained. !!T
	IL_0029: callvirt instance void [mscorlib]System.IDisposable::Dispose()
	IL_002e: ret
} // end of method Program::SafeDispose

```

This code is not expressible in C#. Its rough pseudo-translation would look like:

```cs
static void SafeDispose<T>(ref T d) where T: IDisposable
{
    // d?.Dispose();

    ref T refTemp = ref d;       // create an alias of _variable_ d

    T valTemp;

    // check if T can be null
    if ((object)default(T) == null)
    {
        valTemp = refTemp;      // capture the _value_ of d

        if (valTemp == null)    // if the value is null, we are done
        {
            return;
        }

        refTemp = ref valTemp;  // re-point the alias to the captured value
    }

    // call the member throught the refTemp which points  
    //  - directly to d when T is a struct and cannot be null OR
    //  - to a temporary valTemp containing the captured _value_ of d when T can be null
    refTemp.Dispose();          
}
```

The check ```(object)default(T) == null``` is indeed a dynamic check and may look expensive and potentially boxing. Fortunately most JIT compilers emit separate native method bodies for cases where T is a reference or value types. As a result of such separation the condition becomes a JIT-time constant and the actual native code can omit the check and simply drop the code path that is not applicable as unreachable.  
So, at the end we have correct behavior regardless if T happens to be a class or a struct and performance is good as well.
