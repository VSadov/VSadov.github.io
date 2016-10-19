---
title: Conditional member access operator (thread safety).
tags: [CSharp, Conditional Access, Concurrency]
---
"Is conditional access operator thread-safe?".  
**TL;DR** answer: Yes, it is, but it is an interesting question.

"Is X feature thread-safe?" is often a contextual question and may mean slightly different things. Generally it is a casual way of saying: "Can X be used by multiple threads at the same time?" and fortunately for the conditional member access we can have a simple answer.

One may notice that the C# language spec is for the most part unconcerned about threading. With exception of features specifically dealing with concurrency like ```lock``` or ```volatile```, the semantics of the language is defined in terms of execution of statements and evaluation  of expressions. The thread safety of language features is mostly an emergent property of evaluation strategy and various implementation details.

Examples of guarantees:

- Trivial read of a static variable.  
 Accessing a static variable is mapped to a single read of a static variable as implemented by the underlying platform. CLI standards specify that reads of memory-aligned references and primitives no larger than the native word size is atomic, so reading those is completely thread-safe. That is - the timing of the read would depend on factors such as weakness of the memory model, but you will never read a torn ```int``` with a value that never existed. (assuming that JIT aligns static fields, which it does).
- Trivial read of a local variable.  
 Regular locals are not shared between threads, so it is inherently thread-safe. Local captured into closures is a different matter. If closures are shared between threads then locals can be shared and thus access can have races. If a captured local is larger than the native word, the value may even be subject to tearing.
- Compound operator like ```++``` or ```+=```  
 In ```variable++```, the spec implies a single evaluation of the _variable_, followed by a single read and a single write. So one ```++``` with respect to another concurrent ```++``` is, obviously, not thread safe since two independent reads and two independent writes can interleave and result in lost increments. Interestingly the _variable_ itself is evaluated only once resulting in additional guarantees. For example ```sharedArray[5]+=Val()``` can throw IndexOutOfRange only before evaluating ```Val()```, but not after.  
 Even if ```sharedArray``` is changed by ```Val()``` or changed concurrently while ```Val()``` is running, the write is not supposed to reevaluate ```sharedArray[5]``` and see the change.
 - Iterator methods  
 The only thing that an iterator method does is creating an iterator object in its initial state and there is no sharing implied during that. So as long as all the data involved is not shared or thread-safe for other reasons, it is completely ok to call iterator methods concurrently.
 The iterator object itself, however, is a completely different story. There is clearly a shared state that is mutated on every successful MoveNext. Iterating the same iterator concurrently is a very bad idea.

So, going back to the conditional member access. The language specifies that receiver is evaluated exactly _once_. Based on that, the usage of conditional access operator will not introduce a race into code that does not have it otherwise.

Example:

```cs
static string sharedString;

static int? GetLength() => sharedString?.Length;
```
GetLength will never throw NullReferenceException, even if ```sharedString``` is concurrently assigned ```null```.
The reads and writes of references are atomic, so the single read performed by GetLength will happen either before or after ```sharedString``` is assigned ```null```. There will be no additional reads after the null check so ```.Length``` will either not performed at all or performed on a known non-null value. It should never be applied to ```null```, even in situations involving concurrent mutations.

Another example:

```cs
static Guid? sharedGuid;

static string GetString() => sharedGuid?.ToString();
```
GetString will never throw InvalidOperationException, even if sharedGuid is concurrently assigned ```null```.
That is mostly for the same reasons as above. HasValue property of ```Guid?``` type is governed by a single ```bool``` field and reads and writes of that are atomic.

The whole expression is not thread-safe, however. When you do get non-null results, they could be wildly inconsistent or even represent values of ```sharedGuid``` that never existed. Since ```Guid``` itself is a big and complex structure, its reads and writes are not atomic and subject to tearing. The result could be one half from a value that existed at some point stitched with a half from a different value.
Note that the race here is not introduced by the conditional member access. Simple read of ```sharedGuid``` in the presence of concurrent writes is already not thread-safe.

---

-- Pedantic note:  
In the most general sense, according to [ECMA-335](http://www.ecma-international.org/publications/standards/Ecma-335.htm) standard, only volatile memory accesses are considered sideeffecting. In such model it would be legal to transform

```cs
static Customer customer;

void string Name()
{
  var temp = customer;          //capture the field into a temporary local
  return temp != null ?         //test the temporary
                    temp.Name:  //invoke on a variable that is certainly not null
                    null;
}
```
into

```cs
static Customer customer;

void string Name()
{
  return customer != null ?         //test the variable
                    customer.Name:  //introduce another read
                    null;
}
```

Such "optimization" would introduce a race condition into conditional member access code. The read that compiler generates is not volatile (unless the target variable is a volatile field) and according to [ECMA-335](http://www.ecma-international.org/publications/standards/Ecma-335.htm) can be duplicated.

So why can't the read of the target variable be always a volatile read?  

- Far from all conditional accesses will work with concurrently written/read data. Forcing the access to be volatile could add noticeable overhead on some platforms.
- In reality ECMA-335 memory model is too relaxed to be practically usable and typical CLI implementation follows [CLR2.0 memory model](http://joeduffyblog.com/2007/11/10/clr-20-memory-model/) instead. In that model **reads of shared data cannot be introduced** and "optimization" like above is not permitted.
- Paranoid user can mark ```customer``` as ```volatile``` and make the pattern behave correctly even under ECMA-335 memory model.
