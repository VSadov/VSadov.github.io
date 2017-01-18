---
title: C# Tuples. How tuples are related to ValueTuple.
tags: [CSharp, tuples, Language Design]
---
As a matter of implementation details, C# tuples are implemented on top of ValueTuple types. Here are some details about their relationship.

## What is actually emitted when tuples are used in the code.

Underlying implementation of C# tuples is fairly simple. Tuples of cardinality 2 through 7 are directly mapped to [`ValueTuple`](  https://github.com/dotnet/corefx/blob/15d00331e54e6a2d051c9a939fe1deb72b200e26/src/System.ValueTuple/src/System/ValueTuple/ValueTuple.cs) type of corresponding generic arity. I.E `(int, int)` is represented by `ValueTuple<int, int>`.

```cs  
public void Main()
{
    (int, int) n = (1,1);
    System.Console.WriteLine(n.Item1);
}
```

is emitted as

```cs  
public void Main()
{
    ValueTuple<int, int> n = new ValueTuple<int, int>(1, 1);
    System.Console.WriteLine(n.Item1);
}
```

At 8+ elements things get more interesting. Since arities of `ValueTuple` types go only up to 8, compiler resorts to nesting. The first 7 elements are stored as-is and the rest of elements is stored as a tuple in the `Rest` field of `ValueTuple'8`.

```cs  
public void Main()
{
    var n1 = (1,2,3,4,5,6,7,8);
    System.Console.WriteLine(n1.Item8);

    var n2 = (1,2,3,4,5,6,7,8,9);
    System.Console.WriteLine(n2.Item9);
}
```

is emitted as

```cs  
public void Main()
{
    var n1 = new ValueTuple<int, int, int, int, int, int, int, ValueTuple<int>>(1, 2, 3, 4, 5, 6, 7, new ValueTuple<int>(8));
    System.Console.WriteLine(n1.Rest.Item1);

    var n2 = new ValueTuple<int, int, int, int, int, int, int, ValueTuple<int, int>>(1, 2, 3, 4, 5, 6, 7, new ValueTuple<int, int>(8, 9));
    System.Console.WriteLine(n2.Rest.Item2);
}
```
The encoding scheme used here is recursive. In a 15-element case, the `Item15` will be mapped to `outer.Rest.Rest.Item1`. - I.E. every level of nesting can store 7 elements + remaining tail.

Importantly, the tail is always wrapped in a ValueTuple, even if it is just 1 element. The idea is that if the 8th element is itself a tuple, as in `(int,int,int,int,int,int,int,(int,int))`, the element would be wrapped and thus it could not be confused with a flat tuple that has N more elements, as in `(int,int,int,int,int,int,int,int,int)`.  
This clever encoding scheme is not actually new. It is exactly the same approach as has been used by F# tuples for a long time.

Another interesting observation is that this encoding makes it necessary to have `ValueTuple<T>`, even though by itself 1-element tuples are not expressible in the language.

## What happens if ValueTuple is used in C#7 sources directly?

The backward compatibility requirements dictate that `ValueTuple` structs are allowed in C#7 code, and code that worked in C#6 should continue working in C#7.  
In addition to that, considering that tuples are emitted as `ValueTuple`, the underlying functionality will unavoidably leak through boxing, interop, dynamic, reflection and other scenarios, so why not just make tuple types be "compatible" with the functionality of the underlying types - including fields, properties, methods, implemented interfaces?

There are two ways how this kind of "compatible" could be formalized in C#:  

- exactly the same type.  
Basically it means that the same type has two syntaxes and wherever syntactically possible, one type reference can be replaced with another with no changes to the meaning of the program.    
Example: `System.Nullable<System.Int32>` and `int?` - both refer to exactly the same type  
Anything that can be done with `System.Nullable<System.Int32>` can be done with `int?`.

- identity convertible.  
Here language would track distinct types with different static capabilities, but runtime representation is indistinguishable so a variable of one type can be reinterpreted as a variable of another type.  
Example: `List<dynamic>` and `List<object>`  
`myList[0].Blah()` would work with the first, but would not compile with the second. However you can make an alias of one type to a variable with another.

```cs
public void Main()
{
    // 'lo' is a list of objects
    // this is the only "real" variable we have here
    var lo = new List<object>() {1, 2};

    // 'ld' is an alias to 'lo', typed as a list of dynamic
    // can do this since these types are identity convertible
    //
    // could also pass 'lo' as a 'ref List<dynamic>' parameter
    // but ref locals make example more compact
    ref List<dynamic> ld = ref lo;

    // this compiles with ld
    // GetTypeCode _can_ be called on dynamic
    ld[0].GetTypeCode();

    // this would not compile
    // GetTypeCode _cannot_ be called on object
    error -> lo[0].GetTypeCode();
}
```


So, what happens with tuples?

You can not do `t.Alice` when `t` is typed as `ValueTuple<int, int>`, but can when it is typed as `(int Alice, int Bob)`. Tuple types with element names are clearly separate types. The matters of tuple tuples with element names is worth a separate post, but in short - yes, tuples with element names are **identity convertible** to corresponding `ValueTuple<>` types.

```cs
public void Main()
{
    // the only "real" variable here
    var ii = new ValueTuple<int, int>(1,2);

    // make an alias typed as '(int Alice, int Bob)'
    // can do this since these types are identity convertible
    ref (int Alice, int Bob) ab = ref ii;

    // '(int Alice, int Bob)' has an element 'Bob'
    // it is the same variable as 'ii.Item2'
    ab.Bob = 42;

    // prints 42
    System.Console.WriteLine(ii.Item2);
    // prints 42
    System.Console.WriteLine(ab.Item2);
    // prints 42
    System.Console.WriteLine(ab.Bob);
}  
```

On the other hand the semantical differences between `(int, int)` and `ValueTuple<int, int>` would be so subtle that it was decided to just make them the **same types**. It does mean that `ValueTuple<int, int>` is treated a bit specially by the language. In addition to all the properties common to similar generic types, `ValueTuple<int, int>` would have all the additional functionality of `(int, int)`.

This difference is hard to notice (and that is the point).  
The easiest way is through observing the presence of `ItemN` elements beyond the first 7:  

```cs
public void Main()
{
    // this type matches the pattern of 8-ple (int, int, int, int, int, int, int, int)
    ValueTuple<int, int, int, int, int, int, int, ValueTuple<int>> vt =
        new ValueTuple<int, int, int, int, int, int, int, ValueTuple<int>>
            (1, 2, 3, 4, 5, 6, 7, new ValueTuple<int>(8));

    // surely it has 'Item8' element
    System.Console.WriteLine(vt.Item8);

    // that is actually emitted as
    Console.WriteLine(vt.Rest.Item1);
}
```  

From the implementation prospective, when compiler sees `ValueTuple<>` type whose shape matches an underlying layout of a tuple type, it "upgrades" the type reference to mean the actual tuple type. The transformation applies equally to type references in source as well as in metadata. As a result conforming `ValueTuple<>` types behave exactly as corresponding tuple types.

Overall there are just two distinct groups of tuple types - with element names and without. And the tuple type relationship looks like this:
![Tuple Type relationship]({{ site.url }}/images/Conv.jpg)
