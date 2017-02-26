---
title: C# Tuples. Conversions.
tags: [CSharp, tuples, Language Design]
---
In a statically typed language like C#, every new kind of type or an expression needs to define how it fits into the framework of type conversions. Tuples are not an exception.

Truth be told, initially it was believed that it would be better if tuples have only very limited support for conversions. That was mostly out of fear that composite type such as tuple would run into contradicting scenarios with conversion classification. I.E. if `(int, object)` needs to convert to `(object, int)`, do we have an implicit or explicit conversion or something in between? Is it boxing, or unboxing, or both?  
Forcing the user to deconstruct/reconstruct into a tuple of a desired type would avoid the issues, but it was quickly found to be inconvenient.  

**"Distributing" behavior of tuple conversions.**

The overall guiding principle for tuple conversions is that tuple conversions are composite conversions consisting of `N` underlying conversions, one per element, and classification questions are "distributed" to the underlying conversions, which themselves could be tuple conversions, and in such case the classification is recursive.  

This uncomplicated principle allows tuple conversions to be a relatively low-friction feature, but the design has some interesting details.

**Tuple Literal Conversions and Target Typing.**

C# distinguishes conversions _from expression_ and conversions _from type_.  

Conversions from _expression_ are used when expression results are coerced to be of a particular type. -

```cs
// converion from expression is used to turn int into long
long x = int.MaxValue;  
```

Conversions from _type_ are used in analysis that operates with types - like when determining the best overload resolution candidate.

```cs
void M1(int val) => Console.WriteLine("int");
void M1(long val) => Console.WriteLine("long");

// overload resolution analyses parameter types of the applicable candidates
// M1(int) is selected.
// An implicit conversion from `int` to `long` makes it "better"
M1(short.MaxValue);
```

Conversion from a type generally entails that similar conversion from an expression of such type to the same target type also exists. The opposite is not always true though.  
There are several reasons for the distinction:  

* some expressions do not have natural type at all.  
  _Example:_  `(x)=>x` does not have any type on its own, but converts to `Func<int, int>`
* some expressions have a natural type, but their value fits other types.  
  _Example:_  `42` has type `int`, but implicitly convertible to `byte`, even though `int` does not.
* some types have special behaviors.  
  _Example:_  expressions of type `dynamic` implicitly convert to any type, but the _type_ `dynamic` does not have such conversions.

  Indeed, since most types are implicitly convertible to `dynamic`, having equal conversion the other way would make an overload that takes `dynamic` always ambiguous. An implicit conversion from `dynamic` _expresson_ , though, just means that conversions of `dynamic` values are statically acceptable with the actual dynamic conversion happening at the run time.

Tuple conversions transparently apply same distinctions. There are tuple conversions that exist only for tuple literals, but not for tuple types. That happens when there are conversions from the argument expressions of the literal to the target element types, but not from the types of those arguments.

Example of tuple literal conversions:  

```cs  
// the RHS tuple literal does not have a natural type at all
// because some of the argument expressions do not have a type.
// Yet, it is implicitly convertible to the LHS type
// because every argument _expression_ is implicitly convertible
(Func<int, int>, string, object) t1 = ((x)=>x, null, 1);

// RHS has natural type (int, (int, int)),
// but is implicitly convertible to (byte, (short, object))
// because element-wise implicit conversions from argument expressions exist.
(byte, (short, object)) t2 = (1, (2, 3));
```

**Target-typing and evaluation order**

Conversion of literals to the target type is often called 'target-typing'. That is because the RHS is never materialized in its natural type, instead an instance of the target type is directly created from the RHS value. Indeed, the RHS may not even have a natural type so an instance of such type would not be possible to create.

All the same rules apply to tuple literal conversions, just in a "distributed" manner.

Example:

```cs
(Func<int, int>, string) x = ((x)=>x, null); // is the same as
(Func<int, int>, string) x = ((Func<int, int>)((x)=>x), (string)null);

(byte, short) y = (1, 2);  // is the same as
(byte, short) y = ((byte)1, (short)2);
```

The evaluation order of target typing in tuple literals is observable when both arguments and conversions have sideeffects:

```cs

using System;

class C
{
    static void Main()
    {
        // literal tuple conversion is "distributed" to the arguments of the tuple
        // I.E. every argument is individually target-typed.
        // An instance of (int, int, int) is never created.
        //
        // This is very similar to a constructor call:
        // (C1 a, C1 b, C1 c) t = new ValueTuple<C1,C1, C1>(NextInt(), NextInt(), NextInt());
        //
        (C1 a, C1 b, C1 c) t = (NextInt(), NextInt(), NextInt());

        Console.WriteLine("result: " + t);
    }

    private static int i;
    static int NextInt()
    {
        Console.WriteLine("produced: " + i);
        return i++;
    }
}

class C1
{
    private int val;

    public C1(int val) => this.val = val;

    public static implicit operator C1(int arg)
    {
        Console.WriteLine("converted: " + arg);
        return new C1(arg);
    }

    public override string ToString()
    {
        return val.ToString();
    }
}

=== prints:

produced: 0
converted: 0
produced: 1
converted: 1
produced: 2
converted: 2
result: (0, 1, 2)
```

**Implicit and Explicit conversions**

Naturally, a tuple type/expression:

* has an implicit tuple conversion to the target type if all elements have implicit conversions.
* has an explicit tuple conversion to the target type if all elements have explicit conversion.

Example:

```cs
static void Main(string[] args)
{
    (int, int) ti = (1, 1);

    // type `int` has implicit conversion to `dynamic`, so this works
    (dynamic, dynamic) td = ti;

    // `dynamic` type has _explicit_ conversion to `int`, so this works
    (int, int) ti1 = ((int, int))td;

    // Method((long, long)) is preferred
    // since `(long, long)` is implicitly convertible to `(dynamic, dynamic)`
    // but `(dynamic, dynamic)` has no implicit conversion to `(long, long)`
    Method(ti);
}

static void Method((long, long) ll){}

static void Method((dynamic, dynamic) dd){}
```

The principle is very similar to the lifted conversions in a case of nullable types. As long as `T` converts to `U`, same conversion exists between `T?` and `U?`. The main difference for tuples is that there is more than one underlying conversion and classification of the overall tuple conversion is performed conservatively based on _all_ the underlying conversions.

It is actually possible for a conversion to be both lifted into nullable and into tuple conversions.

Example of a conversion lifted into a nullable, a tuple and then a nullable conversion again:

```cs
(int?, int?)? nubTupleOfNubs = (1, 1);

// `int` has implicit conversion to `long`, thus
// `(int?, int?)? has implicit conversion to `(long?, long?)?`
(long?, long?)? td = nubTupleOfNubs;
```

**Tuple Conversions are Standard Conversions, unconditionally.**

User-defined conversions is, perhaps, the most complicated aspect of C# conversions.

To define composition with user-defined operators, C# language has a concept of Standard Conversions. Standard Conversions are specially privileged conversions - they can "stack" with user-defined conversion _operators_ to form user-defined _conversions_. The reason for the existence of such set of conversions is to widen the applicability of user-defined conversions to more cases than covered by the operator. The reason for the set to be small, an in particular to not include user-defined conversions, is to limit the number of combinations that can result in a conversion.

For example if there is a user-defined conversion operator from type `C1` to `byte`, then an instance of type `C1` is also convertible to `short`. Since there is a standard conversion from `byte` to `short`, compiler can stitch one user-defined operator and one standard conversion into a user defined conversion from `C1` to `short`:

```
C1 --- [implicit user defined operator] ---> byte --- [implicit numeric conversion] ---> short
```

Note that the chain of conversions is never longer than 2 - one user-defined operator and, optionally, one standard conversion on either end. With such constraints the algorithm for finding conversion chains stays fairly simple.

Consider that we are looking for conversion from `T1` to `T2`. Since any user-defined operator involved would need to be defined in either `T1` or `T2`, these are the only types we would look into. We would collect all the user-defined operators defined in these types that convert from `T1` or to `T2`. Now, for those operators that go "half way" - from `T1` to `S1` or from `S2` to `T2`, we would look for a _standard_ conversion that would "complete" the conversion - from `S1` to `T2` or from `T1` to `S2`. If one such found, then we can build a conversion from `T1` to `T2`, if more than one found, then we have an ambiguity.

The point is that the search space has a strict upper bound. If, for example the conversion, that stacks with user defined operator, could be another user-defined conversion, we would need to look at potentially endless chains of conversions involving unlimited number of intermediate types.

The question is whether tuple conversions belong to "The Exclusive Club of Standard Conversions" or not. It was decided that they do.

The convenience is obvious - if, for example, there is a user-defined implicit conversion operator from `C1` to `(int, int)`, then we can implicitly convert `C1` to `(long, long)` as well.

```
C1 --- [implicit user defined op.] ---> (int, int) --- [implicit tuple conv.] ---> (long, long)
```

A curious part is that tuple conversions are standard conversions regardless whether the underlying conversions are standard or not. The underlying conversions could even themselves be user defined conversions.  
This is a case where the conversion classification is _not_ "distributed" to the underlying conversions. Turns out that for the purpose of limiting the search space, such requirement is unnecessary. - at the top we still have a chain of conversions no longer than 2, and underlying element conversions, even if user-defined, cannot nest indefinitely, because tuples cannot nest indefinitely.

It does, however, allow for some interesting scenarios.

Example:  
(implicit expanding into nested tuples of any level)

```cs
using System;
class C
{
    static void Main()
    {
        C1 y = new C1();   

        // `C1` converts to `(byte, C1)`, and thus to `(int, C1)` too.
        // `C1` converts to `(byte, byte)`, and thus to `(int, int)` too.
        // as a result `C1` converts to  types like `(int, (int, ...))`
        // regardless of how deeply they are nested
        (int, (int, (int, (int, (int, (int, (int, (int, (int, (int, (int, int))))))))))) x12 = y;
        System.Console.WriteLine(x12);
    }

    class C1
    {
        private byte x;

        static public implicit operator (byte, C1)(C1 arg)
        {
            return ((byte)(arg.x++), arg);
        }

        static public implicit operator (byte c, byte d)(C1 arg)
        {
            return ((byte)(arg.x++), (byte)(arg.x++));
        }
    }
}

prints:
(0, (1, (2, (3, (4, (5, (6, (7, (8, (9, (10, 11)))))))))))
```

**Tuple conversions and extension methods**

Another interesting example of "distributed" conversion classification in tuples involves applicability checks for extension method receivers.

Generally an expression is acceptable as a receiver of an extension method call if the extension method targets the type of that expression or any of its base types or implemented interfaces.

From a more formal point of view an expression is applicable as a receiver of an extension method if it is convertible to the type of the instance parameter via:

* identity conversion
* implicit reference conversion
* implicit boxing conversion

Based just on that an extension method defined on `object` would be applicable to an expression of type `(int[], int[])`. However an extension defined on `(IEnumerable<int>, IEnumerable<int>)` would not be applicable. Early users of the feature indicated that such limitation is unexpected and inconvenient (see [bug 16159](https://github.com/dotnet/roslyn/issues/16159)).

The solution was to add implicit tuple conversions to the set of allowed instance conversions, but require that all underlying element conversions are valid instance conversions. I.E. the instance conversion rule became distributed and recursive in a case of tuples.

Examples:

```cs

using System;
using System.Collections.Generic;

class C
{
    static void Main()
    {
        // ok,  
        // `string` has implicit reference conversion to `IEnumerable<char>`
        ("hello", "hi").M1();

        // ok
        // `int` has implicit boxing conversion to `object`
        (1, (2, 3)).M2();

        // ok
        // the first element is convertible as a whole
        // the second element is convertible recursively
        (("hi", "hello"), (2, 3)).M2();
    }
}

static class C1
{
    public static void M1(this (IEnumerable<char>, IEnumerable<char>) arg)
    {
        Console.WriteLine("M1");
    }

    public static void M2(this (object, (object, object)) arg)
    {
        Console.WriteLine("M2");
    }
}
```

**Why so complicated?**

Conversions is a pervasive and complicated aspect of the language. Some degree of complexity is unavoidable when a feature needs to behave in a consistent and predictable manner.  

Integration with conversions is often cited as contributing a good portion of the famous "[minus 100 points](https://blogs.msdn.microsoft.com/ericgu/2004/01/12/minus-100-points/)" penalty that applies to every new language feature and needs to be balanced out with benefits.
