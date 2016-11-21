---
title: Conditional member access operator (idiomatic uses).
tags: [CSharp, Conditional Access]
---
Here are some of the less known uses of Null-conditional operator that could be handy to know.

**1. use null-conditional access with void methods**  

A common misconception about null conditional operator is that it can be used only with members that return something. That is probably because of the special treatment where the return type is promoted to nullable (if it cannot represent null already).  
It is, actually, perfectly fine to use null-conditional with a void returning method. The overall expression type is still ```void```, just the method is called conditionally.

```cs  
// get a dictionary if we have one
Dictionary<int, int> dict = GetDictionaryOrNull();

// add something, if we actually have a dictionary
// NOTE: Add has void return type
dict?.Add(42, GetValue());
```

It is particularly nice that execution of the whole```Add(42, GetValue())``` is conditional and thus ```GetValue()``` is only evaluated if ```dict``` is not ```null```.  
Without ```?.``` the same code would not look as nice.

**2. raising events**  

Generally C# events need to be null-checked before invoked. In addition to that, to be safe from races, an event needs to be captured into a local. That is quite a bit of code to just raise an event:

```cs
public class C
{    
    public event Action OnSomething = null;

    public void DoSomething()
    {
        // raise OnSomething, if not null
        var onSomething = OnSomething;
        if ( onSomething != null)
        {
           onSomething();         
        }
    }
}
```  

Raising event, however, is nothing more then invoking ```Invoke``` method on the event. And performing that conditionally on not being null is much clearer with ```?.``` :

```cs
public class C
{    
    public event Action OnSomething = null;

    public void Something()
    {
        // raise OnSomething, if not null
        OnSomething?.Invoke();
    }
}
```

**3. null-conditional and nul-coalescing operator together**   

When receiver of a null-conditional operator is ```null```, the result is also ```null``` of appropriate type. That might be inconvenient in cases where a "default" result, other than ```null``` is supposed to be returned. That can be easily and elegantly fixed by combining ```?.``` and ```??```.  

```cs
// if obj is not null, give me its hashcode or 42 otherwise
int hashcode = obj?.GetHashCode() ?? 42;
```

It may seem that the code has two null checks here, which would be redundant, considering that only one input variable could be null:

```cs
// null check "obj", wrap result in "int?"
int? temp = obj?.GetHashCode();

// null check the "temp", unwrap "int?"
int hashcode = temp ?? 42;
```

Compiler is actually smart enough to understand the meaning of ```?.``` +  ```??``` combination, and emits more optimal code. It knows that the only way for the ```obj?.GetHashCode()``` to be ```null``` is when ```obj``` is ```null``` and in such case the whole expression returns ```42```. When ```obj``` is not ```null```, the result of ```GetHashCode()``` is returned. In fact, there is no need to involve intermediate wrapping/unwrapping of ```int?``` at all.

The actual code that is emitted as equivalent of:

```cs
object stackTemp = obj;
int hashcode = stackTemp != null ? stackTemp.GetHashCode() : 0;
```

**4. use null-conditional in conditions**  

Null-conditional operator has type of ```bool?```, when used with underlying expression of ```bool``` type. Such expression cannot be used directly in conditions. However, there are easy ways to "normalize" the three-state result in to true/false.

When ```null``` should be treated the same as ```false```, use ``` == true ``` or ``` ?? false ```.

```cs
public class C
{        
    // assigned externally
    public static HashSet<int> hs;

    public void Check()
    {		
        if (hs?.Contains(42) == true)
        {
            System.Console.WriteLine("contains");        
        }

        if (hs?.Contains(42) ?? false)
        {
            System.Console.WriteLine("contains");        
        }
    }
}
```

Both ``` == true``` and ``` ?? false``` result in the same code emitted as the resulting conditions indeed have the same semantics. In either case compiler can infer that the only situation in which the condition will be satisfied is when ```hs``` is not ```null``` and when ```hs.Contains(42)``` returns ```true```.  

I personally like ``` == true``` form more, but I have seen ``` ?? false``` used and I find it just as readable.

Again, the intermediate ```bool?```, that would be produced by ```hs?.Contains(42)``` alone, and all expenses related to dealing with it, can be bypassed.

The actual codegen for either of the ```if```s above looks like:

```cs
IL_0000: ldsfld class [System.Core]System.Collections.Generic.HashSet`1<int32> C::hs
IL_0005: dup
IL_0006: brtrue.s IL_000c

IL_0008: pop
IL_0009: ldc.i4.0
IL_000a: br.s IL_0013

IL_000c: ldc.i4.s 42
IL_000e: call instance bool class [System.Core]System.Collections.Generic.HashSet`1<int32>::Contains(!0)

IL_0013: brfalse.s IL_001f

IL_0015: ldstr "contains"
IL_001a: call void [mscorlib]System.Console::WriteLine(string)
```

Which is an equivalent for an optimized:

```cs
HashSet<int> stackTemp = C.hs;
if (stackTemp != null && stackTemp.Contains(42))
{
    Console.WriteLine("contains");
}
```

Conversely ``` != false``` and ``` ?? true``` could be used when ```null``` is to be treated the same as ```true```.

```cs
if (hs?.Contains(42) != false)
{
    System.Console.WriteLine("contains or null");        
}

if (hs?.Contains(42) ?? true)
{
    System.Console.WriteLine("contains or null");        
}
```

In either case, the condition is emitted as an optimized equivalent of:

```cs
HashSet<int> stackTemp = C.hs;
if (stackTemp == null || stackTemp.Contains(42))
{
    Console.WriteLine("contains or null");
}
```


**5. composing null-conditional and lifted operators**  

Null-conditional operator mixes well with lifted operators.

```cs
public class C
{        
    // assigned externally
    public static string s;

    public bool IsLongEnough()
    {		
        return s?.Length * 2 + 1 > 10;
	  }
}
```

Again, compiler knows about the short circuiting nature of ```?.``` and that the only way a ```null``` can get into the calculation is through ```s``` being ```null``` and once that happens we immediately know the result without the need of propagating that ```null``` through the whole chain of lifted calculations.  

The actual code emitted here is equivalent of:

```cs
public class C
{
    public static string s;
    public bool IsLongEnough()
    {
        string stackTemp = C.s;
        return stackTemp != null &&   // <- check for null once
                    stackTemp.Length * 2 + 1 > 10; // <- not null-propagating  
    }
}
```

As you may notice the intermediate nullables are again completely elided by the compiler and the math was simplified to regular (not the null-propagating) form.
