---
title: Why ref locals allow only a single binding?
tags: [CSharp, refs, Language Design]
---
Current restriction on ref locals to be single-assignable is a straightforward and simple way to guard against several potential problems. There are ways to relax the restriction in the future, if that is found to be beneficial enough.

First of all - ref locals are a new kind of ref variables. I have touched some general details common to all ref variables in the earlier posts. [They are not pointers]({% post_url 2016-09-05-refs-not-ptrs %}). They are implemented on top of [managed references]({% post_url 2016-09-17-managed-refs-CLR %}). In many ways ref locals are similar to ref parameters. Both belong to the kind of variables That do not get their own storage and instead are bound to existing storage.

Ref locals are lexically scoped, just like other locals, but the life time of the storage that they are bound to may not match the scopes of the references. That is where things get "interesting".

It could be observed that unrestricted _"any ref local can be bound or re-bound to any variable at any time"_ may lead to the following problems:

**1. If you want to return a ref local, compiler must be able to validate that all possible bindings are "safe to return by ref"**

Here is an example of a ref local that is not safe to return due to ref assignment and nontrivial control flow.

```cs
ref int RotateRefs()
{
  ref var r0 = ref arr[0];  // safe to return
  ref var r1 = ref arr[1];  // safe to return
  ref var r2 = ref arr[2];  // safe to return
  ref var r3 = ref arr[3];  // safe to return

  var local = 42;
  ref var r4 = ref local;   // NOT safe to return !!!

  while(Condition())
  {
    // shift-rotate the refs
    ref var temp = ref r0;
    // (hypothetical syntax for ref re-assignment)
    r0 = ref r1;
    r1 = ref r2;
    r2 = ref r3;
    r3 = ref r4;
    r4 = ref temp;  
  }

  // "r0" is not safe to return here!!!
  // imagine an error message that tries to explain why.
  return ref r0;
}
```

In a most general case, enforcing safe-to-return rule for ref locals would require an analysis similar to the definite assignment analysis. Compiler would need to traverse the control flow graph while propagating what is known about the variables and repeating the analysis until no more knowledge could be gained.

That seems rather complicated, but there is more.

**2. ref local should not be allowed to be used outside of the life times of the possible referents.**

Violating this would lead to variables that are bound to something, that from the point of the language "does not exist".

Consider the following example:

```cs
class Program
{
    static readonly int[] arr = { 0, 0, 0, 0 };

    static void Main(string[] args)
    {
        // bind r0, r1 to something initially
        ref var r0 = ref (new int[1])[0];
        ref var r1 = ref r0;

        for (int i = 0; i < 2; i++)
        {
            // NOTE: possibly initializing "variable" using a value
            //       of its own binding from 2 iterations behind.
            var variable = r0 + 1;

            // keeping the previous binding of "r1" in "r0"
            r1 = r0;

            // binding "r1" to "variable". (hypothetical syntax for ref re-assignment)
            // NOTE: "variable" is about to go out of scope,
            //       but "r0" and "r1" would still be around, bound to what???
            r0 = ref variable;
        }

        // NOTE: both "r0" and "r1" are bound to different
        //       bindings of "variable", which do not exist at this point.
        Console.WriteLine($"Different bindings of 'variable' are equal?: {r0 == r1}");
        Console.WriteLine($"r0 : {r0}");
        Console.WriteLine($"r1 : {r1}");
    }
}
```

As with locals being returned by reference, exposing a local from an inner scope to the code in the outer scopes by the means of a reference is a problem and should be:

1) allowed with undefined behavior.  
2) allowed with exposed locals captured into closures.  
3) disallowed.  

For the same reasons as [with ref returns]({% post_url 2016-10-29-ref-returns-and-locals %}), just disallowing seems more appropriate for C#.

Sadly, _"locals should not be exposed to the outer scopes"_ is a simple principle that is not so simple to enforce. Precise enforcement would require a transitive tracking of all possible bindings of ref locals and validating that scoping rules are not violated at every use point of the references.

For example the following code could be allowed:

```cs
ref int outer;

try
{
    for(;;)
    {
      int inner = 0;

      // binding outer to the inner!!!
      outer = ref inner;

      // assigning inner trough outer.
      // that is ok since we could do it directly here too.
      outer = 42;
    }
}
catch
{
    outer = ref (new int[1])[0];
}

// this is ok also, as long as it is proven that outer
// is not referencing something that is not in scope
outer = 333;
```

The analysis that is required to validate the example above would be something similar to the analysis for the safe-to return validation for ref locals, but the state that is incrementally updated and propagated through control flow paths would be more complex. It would need to reflect the entire collections of all possible scopes from which ref locals could reference variables. The state will be affected by ref-assignments and will have to propagate the state from ref-parameters to the ref returns in a case of method calls.

Note that this analysis would also be sufficient to prove if/when ref locals are safe-to-return. If at particular point, the set of possible scopes for a ref local contains only the "out-of-method" scope, then ref-returning is safe.

Also note that the analysis is very complex, can be computationally expensive and would often yield diagnostics that is hard to act upon. -    
_"ERROR: can't use a ref local here because it is possibly referencing a variable that is out of scope"_, scratch your head and try figure where and how the inconvenient binding was picked up.

Language designers are naturally concerned when a feature requires such analysis. In such situations it is desirable to find a way to constrain the feature in order to reduce the number of scenarios supported, preferably at the cost of uncommon cases. After all, maximizing the number of programs that compile correctly, by itself is not a goal.

In a case of ref locals, it was found that forcing the initialization of ref locals  during initialization and not allowing re-assignment solves the problems described above very nicely:

* the issue with exposing locals by reference to outer scopes is trivially prevented, since it is not possible to initialize something with a variable from an inner scope.  
* safe-to-return property can be simply copied from the initial referent, since we require that there is one and there won't be another.

It is possible that single-assignment requirement will be found too constraining and the language may need to be relaxed a bit. Generally it is ok to start accepting code that used to be an error in previous versions, but the opposite changes are extremely rare.

_Some possible future directions for the feature are:_    

*Addition1:* allow re-assigning of ref locals, but only with something that is safe-to-return, but keep inferring the safe-to-return property from the initial assignment.  
*Addition2:* relax the requirement that ref locals must be initialized at declaration. Treat such locals as safe-to-return, as long as they are definitely assigned.

Note that additions above would allow more code to be legal, primarily when dealing with unscoped/heap variables, while not yet require complicated flow analysis.


**== Pedantic notes:**

Before the ref locals were introduced, it was, in some situations, possible to use `System.TypedReference` in combination with `__makeref`, `__refvalue` keywords as a crude substitute. `TypedReference` clearly contains a managed reference and acts as a proxy, so why all the fuss with C# scoping and why that does not apply to TypedReference?

Well, `__makeref` and `__refvalue` are basically just special keywords that directly map to `makerefany` and `refanyval` IL opcodes in order to provide basic support for `__arglist` feature.  

The functionality is optional in both C# and CLR and is not recommended for general purpose use. In a way, it is a platform/CLR feature (like reflection) and is not very well integrated into the language.

Compiler knows about CLR restrictions of TypedReference - it cannot be boxed, cannot be a field or an array element, cannot be returned from a method, etc... Compiler will try preventing unverifiable code and fatal GC holes, but not much beyond that.

Indeed, the following example shows that `__makeref` completely disrespects scoping rules of C# and does not care if references may outlive the life times of the referenced locals, which, while not fatal, may result in undocumented/unpredictable behavior.

```cs
class Program
{
    static readonly int[] arr = { 0, 0, 0, 0 };

    static void Main(string[] args)
    {
        // bind r0, r1 to something initially
        TypedReference r0 = __makeref((new int[1])[0]);
        TypedReference r1 = r0;

        for (int i = 0; i < 2; i++)
        {
            // NOTE: possibly initializing "variable" using a value
            //       of its own binding from 2 iterations behind.
            var variable = __refvalue(r0, int) + 1;

            // keeping the previous binding of "r1" in "r0"
            r1 = r0;

            // binding "r1" to "variable".
            // NOTE: "variable" is about to go out of scope,
            // but "r0" and "r1" would still be around, bound to what???
            r0 = __makeref(variable);


            // DUMMY NOOP CODE,
            // UNCOMMENT TO CHANGE THE PROGRAM'S OUTPUT!!!!
            //Func<int> dummy = () => variable; // cause "variable" be captured
        }

        // NOTE: both "r0" and "r1" are bound to different
        //       bindings of "variable", which do not exist at this point.
        Console.WriteLine($"Different bindings of 'variable' are equal?: {__refvalue(r0, int) == __refvalue(r1, int)}");
        Console.WriteLine($"r0 : {__refvalue(r0, int)}");
        Console.WriteLine($"r1 : {__refvalue(r1, int)}");
    }
}
```

The output is:  

```
Different bindings of 'variable' are equal?: True  
r0 : 2
r1 : 2
```

And when the noop `Func` is uncommented, the output is:  

```
Different bindings of 'variable' are equal?: False  
r0 : 2
r1 : 1
```
