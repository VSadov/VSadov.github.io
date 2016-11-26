---
title: Local variables cannot be returned by reference.
tags: [CSharp, refs, Language Design]
---
Ability to return by reference introduces an interesting scenario.- What happens when a local variable is returned by reference? Is the variable still alive when its containing method has completed? What happens with the returned reference when callee is invoked again?

These are the questions that every language that allows byref returns needs to answer one way or another. C# design team had to deal with these questions too.

Several options were considered:  

-- **Allow returning locals by reference and leave the behavior unspecified.**  
That is how C++ handles locals returned by reference. Although most C++ compilers would give a warning.  

It is not a viable option for C#. The underlying mechanism for ref returns is [managed pointers]({% post_url 2016-09-17-managed-refs-CLR %}) and those are subject to GC tracking. Regular locals are typically implemented as slots on the stack and subsequent calls will reuse those slots while their local variables may have different types.  
It is extremely dangerous to have a managed reference of type T pointing to unspecified data that has nothing to do with T. If GC attempts to track through such reference and follow what would be the fields of the T instance, but in reality bits and pieces of some other type, it would easily result in heap corruptions.  

Example of a type safety problem if actual stack slots are returned by ref.

```cs

// returns a ref to an Exception local
ref int RefEx()
{
  Exception local = new Exception("hi");
  return ref local;
}

// returns a ref to an int local
ref int RefInt()
{
  int local = 42;
  return local;
}

void TakesTwoRefs(ref Exception s, ref int i)
{
  GC.Collect();
}

void WritesIntIntoEx()
{
  // RefEx will run in the same stack space as RefInt
  // so it is likely that results of RefInt and RefEx
  // would point to the same or overlapping memory location
  // That is already bad by itself.
  // What is worse is that RefInt writes "42" into that location
  // If GC happens during the call, it may see something typed
  // as "Exception" at completely bogus location.
  TakesTwoRefs(ref RefInt(), ref RefEx());
}
```

-- **Extend the life time of the local by allocating it on the heap.**   
This is how Go handles this situation.  

It would not be something entirely new for C#. The approach would be somewhat similar to capturing locals into closures. However, it was decided that in the context of ref returns this is not a good solution.  

*Firstly*, the extent (lifetime) of a local variable in C# matches its scope. Since the caller is running outside of the scope of the callee, then, from its point of view, the locals of the callee _do not exist_. Note that lambdas that cause locals to be captured into closures do not leave the scope, while returning from the method certainly does leave the scope. It would be strange that caller can get an alias to a local that does not exist and perhaps even multiple aliases to multiple incarnation of such local, if caller makes a ref returning call more than once.  
That was not the major point, though. I am sure with some effort such behavior could be rationalized and accepted, if necessary.

*Secondly*, and more importantly, the whole idea of introducing ref returns was motivated by performance-sensitive scenarios where it would allow to avoid redundant copying. Enabling the feature via automatic capturing of locals into display classes would defeat the purpose.

-- **Disallow returning local variables by reference.**  
This is the solution that was chosen for C#. - To guarantee that a reference does not outlive the referenced variable C# does not allow returning references to local variables by reference.  
Interestingly, this is the same approach used by Rust, although for slightly different reasons. (Rust is a RAII language and actively destroys locals when exiting scopes)
