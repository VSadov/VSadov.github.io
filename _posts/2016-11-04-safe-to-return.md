---
title: Safe to return rules for ref returns.
tags: [CSharp, refs]
---
For the reasons explained in the [earlier post]({% post_url 2016-10-29-ref-returns-and-locals %}), C# disallows returning local variables by reference. While the principle of "*Cannot return local variables by reference*" seems very simple, there are many ways for a user to violate the principle directly or indirectly and enforcing it is an interesting challenge.  

So, what is exactly safe to return and what is not?  

Clearly attempting to return a local by reference should trigger an error. But what about a field of a local? If that local happens to be a struct we would be in trouble, since we would still be returning a reference to the local data. On the other hand it would be ok if the local is a class. Therefore there is a need to generalize the rule to include the fields of struct locals as well, recursively. - Field of a field of a field of a field of ... of a local is unsafe to return as long as all types in the chain are structs.

Another interesting question is whether a ref return or a ref parameter themselves are safe to return.

Consider the following example:

```cs
ref int Callee(ref int arg)
{
  return ref arg;
}

ref int Caller()
{
  int s = 42;

  // DANGER!! returning a reference to the local data
  return ref Callee(ref s);
}
```

Here Caller passes a local variable `s` by reference to Callee. Then Callee returns it back by reference. What comes from Caller is essentially `ref s`. While it would be ok for the caller to use that, returning that result by reference would be an equivalent of returning `s` and thus should be prevented.  
In a general case, compiler has no knowledge of what is going on inside Callee. Conservatively, compiler must assume that any byref parameter or its field may be returned back by reference, so as long as any of the ref arguments are not safe to return, the result of the call is not safe to return either.  
Note that in some cases, knowing the types of the ref parameters and the return type, it could be proven that the return can not possibly be referencing data from one of the parameters. However, it was decided to be conservative here for the sake of simplicity and consistency. (considering structs, interfaces and generics, the additional rules could get really complicated).

Here are the actual “safe to return” rules as enforced by the language:

----  
1. **refs to variables on the heap are safe to return**  

2. **ref parameters are safe to return**  

3. **out parameters are safe to return (but must be definitely assigned, as is already the case today)**  

4. **instance struct fields are safe to return as long as the receiver is safe to return**  

5. **“this” is not safe to return from struct members**  

6. **a ref, returned from another method is safe to return if all refs/outs passed to that method as formal parameters were safe to return.**  
Specifically it is irrelevant if receiver is safe to return, regardless whether receiver is a struct, class or typed as a generic type parameter.

----  

The last two rules might look a bit curious. - What's up with "this"?  
The special treatment for "this" was added to handle the following scenario:

```cs

interface IIndexable<T>
{
  public ref T this[int i]
}

ref int First<T>(T arg) where T: IIndexable<int>
{
  // is this safe to return by reference?
  return ref arg[0];
}

```

The problem is that "this" is passed by reference to struct members and by value to class members. If we consider "this" in struct members the same as other parameters for the purpose of rule #6, we would have a problem here since we do not know whether T is a struct or a class. Treating type T conservatively as "can be a struct" would diminish the usefulness of ref returns when used with generics, so another approach was chosen. - "this" is completely ignored at the call site for the purpose of "safe to return" rule as we may not even know whether we are dealing with a struct or a class. To make that safe in cases when we do get a struct, the rule #5 was added. Surely, it is known inside a member whether the container is a struct or a class and the safety can be enforced there.


**Verifiability.**  
There is a little issue with ref returns concerning verifiability. Generally ECMA 335 specifies ref returns as not verifiable. Some JITs are less strict and allow ref returning of heap variables (to accommodate some patterns used by managed c++). That relaxed behavior is still stricter than "safe to return" rules and some examples involving ref returns would only work in scenarios that do not involve formal verification.

It is, however, believed that a system in agreement with "safe to return" rules is actually typesafe and there are plans to add corresponding relaxation to verification rules in the current JITs and tools like PEVerify

It is conceivable that ECMA 335 will be modified or get an implementation specific amendment for such relaxation at some point as well .
