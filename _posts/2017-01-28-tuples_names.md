---
title: C# Tuples. More about element names.
tags: [CSharp, tuples, Language Design]
---
C# tuples can have optional element names. Here are some interesting details about tuple element names and how they are treated by the language.

The matter of allowing named elements was a major choice in the design of C# tuples. It was definitely attractive to allow element names when tuples are used in APIs.

```cs
(int CustomerID, int Orders) GetRecord(){...}
```
is clearly more descriptive and less error prone than

```cs  
// NOTE: the first element is CustomerID, second is Orders
(int, int) GetRecord(){...}
```

On the other hand names could become an obstacle when implementing abstract operations that operate with tuples.  
If a dictionary factory is implemented in terms of Key and Value tuples, would it work with Customers and Orders?

What about completely generic algorithms? -  
If I have `(int X, int Y)` and `int Z`, can I apply the following?

```cs  
(T, U, V) Append<T, U, V>((T, U) tu, V v) => (tu.Item1, tu.Item2, v);
```

If users can't use tuples in generic/abstract scenarios just because they have element names, they'd be inclined to avoid the names altogether making the whole support of names questionable.

C# designers wanted to have both the expressiveness of the names, but also to make sure that names do not "stand in the way" when tuples are used as structural types. So the guiding principle was set to be:  

 **Element names are semantically insignificant except when used directly.**

The tuple types with element names are really the same as ones without. The only difference is the presence of "friendly names".  
In particular all tuple elements have the default `Item1`, `Item2`,.... `ItemN` names. It is allowed to name the elements with their default names, but only as long as they are in the right position.

```cs
// Item2 causes an error here, since it is in a wrong position.
// Item2 name is essentially already taken by the element #2
(int Item1, int X, int Item2) v;
```

Another consequence is that overloaded methods whose signatures differ only in tuple element are disallowed.

```cs  
public class C
{
    public void Ext((int X, int Y) arg){}

    // error CS0111: Type 'C' already defines a member called 'Ext' with the same parameter types
    public void Ext((int V, int W) arg){}
}
```

Conversely - overload resolution will not consider element names when selecting the target of an invocation.  
The following call is ambiguous since, ignoring element names, both `Ext` methods have the same signatures.

```cs  
public class C
{
    public void M()
    {
        var v = default((int X, int Y));

        // error CS0121: The call is ambiguous between the following. . .
        v.Ext();
    }
}

static class Ext1
{
    public static void Ext(this (int X, int Y) arg){}
}

static class Ext2
{
    public static void Ext(this (int V, int W) arg){}
}
```  

**The dynamic type of a tuple variable is just the underlying ValueType.**

Essentially the "tuple" part of these types, including their element names, is just a compile-time decoration that compiler uses and propagates through expressions.

The [erasure](https://en.wikipedia.org/wiki/Type_erasure) of tuple related information can be easily observable by checking the type of boxed instances or the static type as tracked by  CLR type system.

```cs
class Program
{
    static void Main(string[] args)
    {
        // tuple instances do not know they are tuples
        object instance = (Alice: 1, Bob: 2);
        System.Console.WriteLine(instance.GetType());

        // CLR does not trace tuple types either.
        PrintStaticType((Alice: 1, Bob: 2));
    }

    static void PrintStaticType<T>(T arg)
    {
        System.Console.WriteLine(typeof(T));
    }
}

The output is:

   System.ValueTuple`2[System.Int32, System.Int32]
   System.ValueTuple`2[System.Int32, System.Int32]
```
**Representing element names in metadata**

Since CLR types themselves do not store tuple information, compiler puts extra information to specify tuple element names in member signatures.  
The encoding is rather simple - `TupleElementNamesAttribute` simply contains an array of element name strings in the pre-order depth-first traversal order.

```cs
// "C" and "F" are intentionally missing - will be encoded as "null" strings.
static Dictionary<(int A, int B), (int, int D)?> Test((int[] E, int)[] arg)
{
    return null;
}
```

Emitted as an equivalent of:  

```cs

[return: TupleElementNames(new string[]{"A","B",null,"D"})]
private static Dictionary<ValueTuple<int, int>, ValueTuple<int, int>?> Test
    ([TupleElementNames(new string[]{"E",null})] ValueTuple<int[], int>[] arg)
{
    return null;
}
```

As explained in [earlier post]({% post_url 2017-01-16-tuples_valuetuple %}), ValueTuple types that matches a tuple pattern are promoted into tuple types during metadata import. In addition to that, the element names are "rehydrated" from a TupleElementNames attribute, if one is specified for the given part of a member signature.  

Note that in terms of cross-language interoperability, understanding `TupleElementNames` attribute or the tuple encoding pattern is optional.  
If the consuming language does not care about element names (like F#) it can ignore the attribute and just see the signature with "nameless" tuples. If the consuming language does not understand tuples at all (like C#6) it can still interoperate by using ValueTuple types.

**Compile time propagation of tuple types**

Note that compile time propagation of the tuple types can go quite far, including through the generic type inference. At compile time the tuple types are "real types".

Example of a tuple type with element names propagated through several level of type inference.

```cs
static void Main(string[] args)
{
    // The only argument with a natural type is the "42"
    // T infers its type from "42"
    // U has dependency on T which is resolved via lambda inference once we know T
    // U[ ] is the return type of Apply and is known once we know U
    // type of 'r' is inferred to be the same as the return type of 'Apply'
    var r = Apply(42, (val) => (Alice: val, Bob: val.ToString()));

    // As a result
    // r has type: (int Alice, string Bob)[ ]
    Console.WriteLine(r[0].Alice);
    Console.WriteLine(r[0].Bob);
}

static U[] Apply<T, U>(T arg, Func<T, U> f)
{
    return new U[] { f(arg) };
}
```  

The element names are not always involved in the inference. In scenarios where tuple arguments match tuple parameters of the same cardinality, the inference works in a purely structural way and element names are ignored.  

Surely, when type parameters are inferred from the argument element types, the names cannot take part in that.

```cs
static void Main(string[] args)
{
    var v = (Alice: "hi", Bob: "there");

    // T is inferred to be 'string'
    // so is the type of r
    var r = Test(v).Result;
    Console.WriteLine(r.ToUpper());

    Console.WriteLine(Append(t: (1, 2), third: 3));
}

// T is inferred from the first element type of the argument tuple
static async Task<T> Test<T, U>((T, U) arg)
{
    // just await something
    await Task.Yield();

    return arg.Item1;
}

// T, U are inferred from element types of 2-tuple argument
// and used as element types of 3-tuple result
// element names are unrelated and unimportant for inference purposes here
static (T First, U Second, V Third) Append<T, U, V>((T First, U Second) t, V third)
{
    return (t.First, t.Second, third);
}
```

**Tuple type merging and dropping of element names**

When inferring tuple names from multiple sources, a situation may arise where multiple names for the same element would be inferred. In such case these names are "dropped" leaving the corresponding tuple element unnamed.

Indeed, there are only two design choices here - drop conflicting names or make the whole scenario an error. However making it an error would contradict the idea that presence of element names is semantically insignificant.

```cs
static void Main(string[] args)
{
    var x = (Alice: "hi", Bob: "there");
    var y = (Alpha: "bye", Beta: "bye");

    // T is inferred to be
    //    (string Alice, string Bob)  and also
    //    (string Alpha, string Beta)
    //
    // To resolve apparent ambiguity conflicting names are dropped.
    // T is just: (string, string)
    var z = OneOrAnother(x, y, DateTime.Now.DayOfWeek == DayOfWeek.Friday);

    // this would be an error
    // Console.WriteLine(z.Alice);

    // this is still ok
    Console.WriteLine(z.Item1);

    var x1 = (Alice: "bye", Todd: "bye");

    // only ambiguous names are dropped
    // z1 has type:  (string Alice, string)
    var z1 = DateTime.Now.DayOfWeek == DayOfWeek.Friday ?
                    x :
                    x1;

    // this is ok
    Console.WriteLine(z1.Alice);

}

// T is inferrable from both x and y
static T OneOrAnother<T>(T x, T y, bool flag)
{
    return flag ? x : y;
}
```

**Can element names become "semantically significant" through lambda inference?**

There is an interesting scenario which seemingly demonstrates that element names _can_ have effect on overload resolution when combined with lambda inference. The example below is able to steer overload resolution to one of the candidates by using specific tuple element names.  
However at closer examination, the element names are actually _used directly_ in this scenario, so of course they make a difference. It is not a case where two tuple types compete for better applicability, it is a case where two reified lambdas compete, and one would not compile.

```cs
static void Main(string[] args)
{
    // calls the first Select -
    // the only case where ".Bob" would not be an error
    var r = Select(1, 2, t => t.Bob);

    // ambiguity error: lambda can be applied in either case
    // var r1 = Select(1, 2, t => t.Alice);
}

delegate TResult Selector1<TArg, TResult>(TArg arg);

static T Select<T>(T x, T y, Selector1<(T Alice, T Bob), T> selector)
{
    Console.WriteLine("first overload");
    return selector((x, y));
}

delegate TResult Selector2<TArg, TResult>(TArg arg);

static T Select<T>(T x, T y, Selector2<(T Alice, T Todd), T> selector)
{
    Console.WriteLine("second overload");
    return selector((x, y));
}
```

**Diagnostics on element name mismatches**

Considering how easily element names can be cast aside, the language designers had concerns that compiler would be less than helpful against certain kinds of mistakes. Clearly some name mismatches could be indicative of a confusion or a typo.

```cs
static void Main(string[] args)
{
    // Warning!!
    //
    // "Boook" is ignored. Likely a typo.
    M1((Boook: 1, Chapter: 2));

    // Warnings!!
    //
    // "First" and "Last" are mismatched causing both to be dropped.
    // That is highly suspicious
    var r = DateTime.Now.DayOfWeek == DayOfWeek.Friday ?
                (ID: 1, First: "F", Last: "L") :
                (ID: 2, Last: "L", First: "F");

}

static void M1((int Book, int Chapter) arg)
{
    // . . .
}
```

While language is pretty clear on the semantics of the above samples, the code is likely to be unintentional.  

Determining scenarios that result in warnings is not an easy task. The scenarios must be much more likely a result of an error than not. In addition there should be reasonable and obvious ways to fix the violations. In the initial release the warnings are produced under the following conditions:

- It is an identity conversion from a tuple literal.
- Some names specified in the literal are dropped as a result of conversion.

The mistake in such scenarios is fairly clear - the name is explicitly specified and immediately ignored due to mismatch - that is at very least redundant. The most trivial fix is to just fix the name to match destination or to remove it entirely.

There are plans to improve the name mismatch analysis. Some of those plans are captured in this [WorkItem](https://github.com/dotnet/roslyn/issues/14217). More data/statistics on the real-world use of tuples would be useful to improve the analysis as well.

**Element names must match when overriding or implementing.**

Some language designers felt particularly strong about overriding and implementing scenarios. There was some discussion whether changing element names upon overriding/implementing is a bad enough pattern that it must be a compile error or just a warning.  
What tipped the scales towards making this an error is that if error is found to be too strict, it can be relaxed, without being a compatibility issue. Change in the opposite direction would be breaking.  

```cs
static void Main(string[] args)
{
    Animal a = new Dog();

    a.M1(). //  AnimalName or DogName ?
}

abstract class Animal
{
    public abstract (int ID, string AnimalName) M1();
}

// Changing element names when overriding could be confusing to the caller.
class Dog: Animal
{
    // Error: cannot change tuple element names when overriding.
    public override (int ID, string DogName) M1()
    {
        return (1, "Spot");
    }
}
```

Note that these restrictions get validated and reported _after_ the semantic analysis. The element names are ignored while determining overriding/implementing relationships, but when it is done, it is enforced that element names match.
