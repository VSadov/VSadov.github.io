---
title: What happens if 'out' parameter is not assigned by the calee?
tags: [C#]
---
C# specification is vague on scenarios that cannot possibly happen in a closed C#-only system. Assuming the compiler does not have bugs, ```out``` protocol is supposed to be guaranteed by both the caller and the callee. The situations where rules are not followed lead to "unspecified" behavior. How bad can it get though? 

C# ```out``` calling convention is built on mutual _trust_. ```out``` parameter is essentially a ```ref``` parameter that callee promises to assign by marking with ```out``` attribute. Broken promise here primarily affects definite analysis and features based on that analysis.

Since C# cannot break the protocol, lets use another language that can. 
VB to the "rescue".

```VB
Imports System.Runtime.InteropServices

Public Class Class1
    Public Shared Sub Foo(<Out> ByRef x As Integer)
        ' Bah, could not be bothered to assign x
    End Sub
End Class

``` 

And the client will be like:

```cs
static void M()
{
    while (true)
    {
        int x;

        ClassLibrary1.Class1.Foo(out x);
        System.Console.Write($"{x++} ");
    }
}
```

The output is
```
0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 . . .
```

In C# model all variables are supposed to be created anew when entering the containing scope. We are supposed to get a new ```x``` every time. In reality compiler reuses the same slot for ```x``` and relies on definite analysis to ensure that you will never observe the value left by the previous iteration. The analysis proves that the variable is guaranteed to be overwritten by ```Foo``` so the program compiles with undefined behavior.

So far it does not look too bad. Yes, we see the values from the previous iteration, but the behavior seems deterministic.
Well, that is just because we are lucky and ```x``` is always stored in the same location. 

Lets try something more complex.

```cs
static async Task M()
{
    while (true)
    {
        int x;

        await AwaitsSometimes();

        ClassLibrary1.Class1.Foo(out x);
        System.Console.Write($"{x++} ");
    }
}

private static Random random = new Random(42);
static async Task AwaitsSometimes()
{
    if (random.Next() % 2 == 1)
    {
        await Task.Yield();
    }
}
```

Now behavior is:

Release build: 

```
0 0 1 2 3 4 0 0 0 1 2 3 4 0 0 1 0 0 0 0 0 0 0 1 2 0 0 1 0 1 2 3 0 1 2 0 . . .
```

Debug build: 

```
0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 . . . 
```  

What is going on?

When ```AwaitsSometimes``` randomly awaits, it forces the caller to suspend and schedule the rest of itself as an asyncronous continuation of ```Task.Yield```. While doing that, caller saves the values of the local variables in a display storage, so that when resumed, it could continue with the same state. It does not, however, save the value of ```x``` considering that pointless. For all it knows from the definite assignment analysis, at the point of ```await AwaitsSometimes()```, ```x``` contains undefined junk that will be overwritten before it is used anyways.
Since we go into suspension _sometimes_, we observe the old value _sometimes_. Not the behavior you want from a program...

Why then Debug behavior is different?

In Debug builds compiler forces that all locals in the same scope as ```await```, live in the display storage without consulting with definite assignment analysis. That is done to make debugging experience better. When resuming after await, _all_ locals will keep their values and can still be examined in the debugger. When values of most of the locals are not used after await, such practice could be very wasteful.
That makes us "lucky" again in Debug, but Release shows that behavior is truly undefined.

Garbage in, garbage out. Do not break promises.
