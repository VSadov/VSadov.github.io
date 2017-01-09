---
title: C# Tuples. Why mutable structs?
tags: [CSharp, tuples, Language Design]
---
C# defines tuples as mutable value types. Considering the general guidance against mutable structs, it may look as a peculiar design choice.

Indeed why? And in particular, why existing family of `System.Tuple<>` classes did not fit?

## Why value types.

As with many other features, design of C# tuples started with investigating existing pain points that the new feature was supposed to fix. A special interest was paid to the existing tuple-like types in existing code bases. By tuple-like, here I mean an abstract datatype used just to bundle up several values without giving them any particular meaning.

Turns out Roslyn itself had three!! unrelated custom tuple-like types. There were `ValueTuple<T1, T2>`, `ValueTuple<T1, T2, T3>` and so on. There was `Pair`. And at some point, I believe, there was `StructTuple`, which was later merged with `ValueTuple`. In addition to that there was some use of `KeyValuePair` for purposes that have nothing to do with either keys or values - just to combine two unrelated pieces of data and to pass around. Other projects, including .Net FX itself, had similar finds. (for example [`Pair`]( https://github.com/dotnet/corefx/blob/fb0a1d90d874eba4d1b00227d31009496f67002d/src/System.Linq.Parallel/src/System/Linq/Parallel/Utils/Pair.cs))

The common theme for all these was trafficking multiple pieces of data as a single unit and in particular returning multiple values from methods. One existing solution for returning multiple values is through `out` parameters, but it is often inconvenient and does not work for `async` methods at all, so an aggregate type is needed.  
On the other hand programmers are not compelled to create specialized types just to move 2+ items around, so they create generalized helper types like `Pair` that otherwise have no special meaning - just to hold two things. These are scenarios where tuples would step in.

It was also observed that these types are typically structs. Considering that a tuple is just a combination of values with no identity on its own, the value semantics of structs appeared to be convenient.  
Ideally it would be possible to just push more than one value to the stack upon returning from a method, but since that is not possible, pushing a single struct containing those values is the next best thing. Using classes here would only add the costs of allocation and indirection.

At the end it appeared that structs are a sufficient and also cheap solution to combining multiple piece of data into a single unit.  
The only consideration in the favor of classes was the cost of copying in case if tuples are large. Turns out the large tuples, while possible, are exceedingly rare.

## Why Mutable

Since tuples are structs there is no much point in making them immutable. The readonliness of a struct is a property of the whole variable. If a tuple variable is assignable, it can be changed to contain any value regardless of the readonliness of individual fields.

Example:

```cs
struct ImmutableTuple<T1, T2>
{
    // immutable, right?
    public T1 Item1{get;}
    public T2 Item2{get;}

    public ImmutableTuple(T1 item1, T2 item2)
    {
        Item1 = item1;
        Item2 = item2;
    }

    public override string ToString() => $"({Item1}, {Item2})";
}

static void Test(ImmutableTuple<int, int> arg)
{
    // change arg arbitrary to (42, 42)
    arg = new ImmutableTuple<int, int>(42, 42);
    System.Console.WriteLine(arg);
}
```

Interestingly, in the example above compiler is not even trying to create a new instance and assign. It directly initializes `arg` with new values since it knows there is no difference:

```cs
void Test (
  valuetype Program/ImmutableTuple`2<int32, int32> arg
) cil managed
{
// Method begins at RVA 0x206b
// Code size 23 (0x17)
.maxstack 8

// load the address of "arg"
IL_0000: ldarga.s arg

// load values
IL_0002: ldc.i4.s 42
IL_0004: ldc.i4.s 42

// call the constructor "in-place" on the arg
IL_0006: call instance void valuetype Program/ImmutableTuple`2<int32, int32>::.ctor(!0, !1)

IL_000b: ldarg.0
IL_000c: box valuetype Program/ImmutableTuple`2<int32, int32>
IL_0011: call void [mscorlib]System.Console::WriteLine(object)
IL_0016: ret
} // end of method Program::Test
```

Surely, the cheapest instance is the one that was not created at all.

Anyhow, immutability of tuples would not prevent elements from being changeable. It would only serve to annoy users when they want to change and have to do it in a roundabout way instead of directly assigning.

Also, once tuples are mutable, there is no point in using properties - that would just prevent passing individual elements by reference, resulting in unnecessary diminished functionality compared to using two variables directly.
Indeed - if you had two mutable variables, you could pass either by reference, why prevent that once you have them in a mutable tuple?

The end result - **_C# tuples are extremely lightweight constructs - they are structs and their elements are mutable and directly addressable._**

```cs
using System;

public class C
{
    public static void Main()
    {
        // tuple1 is a struct
        var tuple1 = (Alice: 1, Bob: 2);

        // tuple2 is a copy
        var tuple2 = tuple1;

        // elements can be assigned
        tuple1.Alice = 42;

        // elements can be passed by reference
        Inc(ref tuple1.Bob);

        // tuple2 is indeed a copy. (prints "False")
        System.Console.WriteLine(tuple1.Equals(tuple2));
    }

    public static void Inc(ref int x)
    {
        x++;
    }
}
```


## Note aside: So, is it all that useless to make the content of a struct readonly?

Not at all !!!

While design of a struct, on its own, cannot prevent its values from being assigned as a whole, it can prevent piece-wise assignment. That is very important if a given struct has internal invariants guaranteed by the construction.

Consider the following struct:

```cs

// text span starting at "Start" and ending at "End"
struct TextSpan
{
     private readonly int start;
     private readonly int end;

     public TextSpan(int start, int end)
     {
          if (start < 0) throw new ArgumentException();
          if (end < 0) throw new ArgumentException();
          if (end < start) throw new ArgumentException();

          this.start = start;
          this.end = end;
     }

     // none of the following can be negative!!
     public int Start => start;
     public int End => end;
     public int Length => end - start;
}
```

Note that `TextSpan` values have special meaning. There are also guarantees that they have nonnegative `Start` and `End` and nonnegative `Length`. Even if a struct can be overwritten as a whole by another value, that new value would still have the same guarantees.  
In a normal program (I.E. no racing assignments, use of reflection or unsafe code) the invariants of `TextSpan` would hold regardless of how it is used. Thanks to it being immutable!!!

What makes tuples different here is that tuples do not guarantee any invariants - they are just containers, and any values are allowed, so nothing extra would be achieved by being immutable.
