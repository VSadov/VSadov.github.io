---
title: C# Local Functions vs. Lambda Expressions.
tags: [CSharp, Local Functions, Lambda Expressions, Language Design]
---
C# Local Functions are often viewed as a further enhancement of lambda expressions. While the features are related, there are also major differences.  

Local Functions is the C# implementation of [Nested function](https://en.wikipedia.org/wiki/Nested_function) feature. It is a bit unusual for a language to get support for nested functions several versions after supporting lambdas. Usually it is the other way around.

Lambdas, or first-class functions in general, require implementation of local variables that are not allocated on the stack and have life times tied to the functional objects that need them. It is nearly impossible to implement them correctly and efficiently without relying on Garbage Collection or dropping the burden of variable ownership on the user via solutions such as capture lists. That was a serious blocking issue for some early languages.  
A simple implementation of nested functions does not run into such complications, so it is more common for a language to support only nested functions and not lambdas.

Anyways, since C# had lambdas for a long time, it does makes sense to look at the Local Functions in terms of differences and similarities.

## Lambda expressions.##

Lambda expressions like `x => x + x` are expressions that abstractly represent a piece of code and how it binds to parameters and variables in its lexical environment. Being an abstract representation of code, a lambda expression cannot be used on its own. In order to use values produced by a lambda expression, it needs to be converted to something more material such as a delegate or an expression tree.

```cs
using System;
using System.Linq.Expressions;

class Program
{
    static void Main(string[] args)
    {
        // can't do much with the lambda expression directly
        // (x => x + x).ToString();  // error

        // can assign to a variable of delegate type and invoke
        Func<int, int> f = (x => x + x);
        System.Console.WriteLine(f(21)); // prints "42"

        // can assign to a variable of expression type and introspect
        Expression<Func<int, int>> e = (x => x + x);
        System.Console.WriteLine(e);     // prints "x => (x + x)"
    }
}
```

There are several things that are worth noting:  

- lambdas are expressions that produce functional values.

- lambda values have unbounded life times - from the execution of the lambda expression and as long as any reference to the value exists. That implies that any local variables used, or "captured", by the lambda from the enclosing method must be allocated on the heap. Since the life time of the lambda value is not limited by the life time of the stack frame where it was produced, the variables cannot be allocated on that stack frame.

- lambda expression requires that all external variables used in the body are definitely assigned at the time the lambda expression is executed. The moment of the first and the last use of a lambda are rarely deterministic, so the language assumes that lambda values can be used right after creation and as long as they are reachable.  
As a result a lambda value must be fully functional at the point of its creation and all outer variables that it uses must be definitely assigned.

```cs
        int x;

        // ERROR: 'x' is not definitely assigned
        Func<int> f = () => x;
```

- lambdas do not have names and cannot be referred to symbolically. In particular lambda expressions cannot be declared recursively.  

NOTE: It is possible to _make_ a recursive lambda by invoking a variable to which the lambda is assigned or by passing to a higher-order method which self-applies its parameter (see: [Anonymous Recursion in C#](https://blogs.msdn.microsoft.com/wesdyer/2007/02/02/anonymous-recursion-in-c/)), but that does not make such expressions truly self-referential.

## Local functions. ##

Local function is basically just a method declared inside another method as a way of reducing visibility of the method to the scope within which it is declared.

Naturally, the code in a local function has access to everything that is accessible in its containing scope - local variables, enclosing methods's parameters, type parameters, local functions. A notable exception is the visibility of outer method's labels. Labels of the enclosing method are not visible in a local function. That is just normal lexical scoping and it works the same as in lambdas.

```cs  
public class C
{
    object o;

    public void M1(int p)
    {
        int l = 123;

        // lambda has access to o, p, l,
        Action a = ()=> o = (p + l);
    }

    public void M2(int p)
    {
        int l = 123;

        // Local Function has access to o, p, l,
        void a()
        {
          o = (p + l);
        }
    }
}
```

The obvious difference from lambdas is that local functions have names and can be used without any indirection. Local functions can be recursive.

```cs
static int Fac(int arg)
{
    int FacRecursive(int a)
    {
        return a <= 1 ?
                    1 :
                    a * FacRecursive(a - 1);
    }

    return FacRecursive(arg);
}
```

The main semantical difference from lambda expressions is that local functions are not expressions, they are declaration statements. Declarations are very passive entities when it comes to code execution. In fact declarations do not really get "executed". Similarly to other declarations like labels, local function declarations simply introduce the functions into containing scope without running any code.  

What is more important is that neither declarations by themselves nor regular invocations of a nested function result in an indefinite capture of the environment. In simple and common cases, like an ordinary invoke/return scenario, the captured locals do not need to be heap-allocated.

Example:

```cs
public class C
{    
    public void M()
    {
        int num = 123;

        // has access to num
        void  Nested()
        {
           num++;
        }

        Nested();

        System.Console.WriteLine(num);
    }
}
```

The code above is emitted as roughly equivalent of (decompiled):

```cs
public class C
{
  // A struct to hold "num" variable.
  // We are not storing it on the heap,
  // so it does not need to be a class
  private struct <>c__DisplayClass0_0
  {
      public int num;
  }

  public void M()
  {
      // reserve storage for "num" in a display struct on the _stack_
      C.<>c__DisplayClass0_0 env = default(C.<>c__DisplayClass0_0);

      // num = 123
      env.num = 123;

      // Nested()
      // note - passes env as an extra parameter
      C.<M>g__a0_0(ref env);

      // System.Console.WriteLine(num)
      Console.WriteLine(env.num);
  }

    // implementation of the the "Nested()".
    // note - takes env as an extra parameter
    // env is passed by reference so it's instance is shared
    // with the caller "M()"
    internal static void <M>g__a0_0(ref C.<>c__DisplayClass0_0 env)
    {
        env.num += 1;
    }
}
```

Note that the code above calls the implementation of "Nested()" directly (not via a delegate indirection) and does not introduce an allocation of display storage on the heap (as lambda would have). The locals are stored in a struct instead of a class. The life time of the `num` was not altered by its use in `Nested()`, so it can still be allocated on the stack. `M()` could just pass `num` by reference, but compiler uses a struct for packaging, so that it could pass all locals like `num` using just one env parameter.

Another interesting point is that Local Functions can be used as long as they are visible in a given scope. This is an important fact that makes recursive and mutually recursive scenarios possible. That also makes the exact location of the local function declaration in the source largely unimportant.

For example all the variables of the enclosing method must be definitely assigned at the _invocation_ of a Local Function that reads them, not at its declaration. Indeed, making that requirement at declaration would not do any good if an invocation can happen earlier.

```cs
public void M()
{
    // error here -
    // Use of unassigned local variable 'num'
    Nested();

    int num;

    // whether 'num' is assigned here or not is irrelevant
    void  Nested()
    {
       num++;
    }

    num = 123;

    // no error here - 'num' is assigned
    Nested();

    System.Console.WriteLine(num);
}
```

Also - if a local function is never used, it is no better than a piece of unreachable code and any variable, that it would otherwise use, does not need to be assigned.

```cs
public void M()
{        
    int num;

    // warning - Nested() is never used.
    void  Nested()
    {
       // no errors on unassigned 'num'.
       // this code never runs.
       num++;
    }
}
```

## So, what is the purpose of Local Functions? ##

The main value proposition of local functions, in comparison to lambdas, is that local functions are simpler, both conceptually and in terms of run time overhead.

Lambdas serve their role as a [first-class functions](https://en.wikipedia.org/wiki/First-class_function) very well, but sometimes you only need a simple helper. Lambda assigned to a local variable could do the job, but there is an overhead of indirection, allocation of a delegate and possibly a closure. A private method works too and is cheaper to call, but there is an issue with encapsulation, or lack thereof. Such helper would be visible to everybody in the containing type. Too many helpers like this can result in a serious mess.

A Local Function fits this scenario nicely. The overhead of calling a Local Function is comparable with a call to a private method, but there is no issue with polluting the containing type with a method that nothing else should call.
