---
title: ref returns are not pointers.
tags: [CSharp, refs]
---
With the introduction of ref returns and ref locals in C#7, I often hear questions that indicate confusion between byref returns and pointers.

* Why ```ref void Foo()``` is not allowed?
* Can ref return be null? (not the referent, but the ref itself). How do I test for nulls?
* Why no operations to increment refs - like to point to the next array element? Can we have indexing operations on refs?  
* Is it possible to have a ref of a ref?

What points to confusion is that in a signature like ```ref int foo()``` the ```ref int``` is not a meaningful isolated part. The ```ref``` modifier is just saying that the result of the _whole_ method is passed by reference. The method still returns an ```int```.

Interestingly, ```ref``` is definitely not a new thing in C#. ```ref``` parameters were in the language since the v1.0. Somehow the ref parameters are not causing questions above. Perhaps it is easier to imagine that ```ref``` in ```foo(ref int x)``` just specifies _how_ ```int x``` is a passed. However when we see ```ref int``` describing a return of a method, it is possible to see it as a method returning something typed ```ref int```. It does not. From the language prospective, the return type is still an ```int```.

Consider the following examples:

- Overload resolution

```cs
void foo(long x){...}
void foo(int x){...}

ref int bar(){...}

// overload resolution selects foo(int) here
// since the type of bar() is int
foo(bar());
```

- Generic type inference

```cs
T foo<T>(T arg){...}
T bar<T>(ref T arg)

ref int baz(){...}

// T is inferred to be an int.
// The part that baz returns by reference is irrelevant
foo(baz());

// same here. T is an int.
// The part that baz returns by reference only
// allows us to pass baz further by reference
bar(ref baz());
```


As we see above, ```ref int``` type does not really exist.
Just like ```readonly``` in a field declaration, ```ref``` , in both parameter and return cases, does not apply to the type. It applies to the variable.

Similarly with ```ref``` locals -

```cs
ref int x = ref arr[1];
```

```x``` has type ```int```. ```ref``` just says that ```x``` does not have its own storage. It is an alias of an existing variable - ```arr[1]``` in this case.

More examples:

```cs
ref int x = ref arr[1];

var y = x;          // y is an int and has the same _value_ as x
ref var z = ref x;  // z is an int and represents the same _location_ as x
```

Coming back to the questions.

* Why ```ref void Foo()``` is not allowed?  
Because Foo() does not return anything. It can't do that by reference.
* Can ref return be null? (not the referent, but the ref itself). How do I test for nulls?  
Same as "can a variable be null? Not what is stored in the variable, but variable itself." - Not possible. Not, unless something is fundamentally broken.
* Why no operations to increment refs - like to point to the next array element? Can we have indexing operations on refs?  
Operations are specific to the type of variables. And ```ref``` is not a part of the type.
* Is it possible to have a ref of a ref?  
Since ```ref int``` is not something meaningful on its own, ```ref ref int``` is not either.  

Note how this all is different from ```int *``` which is an actual type. You can declare variables of type ```int* x```, and operations specific to pointers (dereference, indexing, incrementing...) will be applicable to the variables.

There are no byref types in C# though.
