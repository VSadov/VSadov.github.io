---
title: Conditional operator (capturing the receiver).
tags: [CSharp, Conditional Operator]
---
Conditional access operator is a popular new language features that was added in C#6. At the high level it is an operator that replaces the boiler plate code of conditionally invoking a member off a variable that can be null. But there is more.

The typical code that conditional operator replaces looks like:

```cs
static Customer customer;

static string Name()
{
  return customer != null ?
                    customer.Name:
                    null;
}
```

Now you can write:

```cs
static Customer customer;

static string Name()
{
  return customer?.Name;
}
```

The interesting point here is that the variant with conditional access is not just shorter and more readable. It is also avoiding subtle double-evaluation flaw.
If you look carefully at the first example, there are two reads of ```customer``` field. The first read is to check for null and the second is to invoke the ```Name```. If there is a possibility that ```customer``` can be assigned ```null``` between the null check and the invocation, it would result in occasional NullReferenceException, - negating the whole point of the null check.

The actual code emitted for the conditional access is more similar to:

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

It may seem like the usage of conditional access could result in a lot of extra locals. However that is generally not a problem. The temporary only needs to exist "logically". In many cases compiler can achieve the same results by simply duplicating the value on the VES execution stack.

The actual IL for the code above looks like:

```cs
.method private hidebysig static
	string Name () cil managed
{
	// Method begins at RVA 0x206b
	// Code size 17 (0x11)
	.maxstack 8

	IL_0000: ldsfld class Customer Program::customer  // read the field _once_
	IL_0005: dup                   // <--- duplicate the value
	IL_0006: brtrue.s IL_000b

	IL_0008: pop
	IL_0009: ldnull
	IL_000a: ret

	IL_000b: call instance string Customer::get_Name()
	IL_0010: ret
} // end of method Program::Name
```

Also note that the compiler is reasonably smart about the cases where the temporary is needed. If ```customer``` happens to be a variable only accessible from the current method - like a byval parameter, it will not create the temporary.

```cs
static string Name(Customer customer)
{
    return customer?.Name;
}
```

The corresponding IL will not make a copy. Compiler knows that byval argument cannot change between the null check and the invocation. There are no observable sideeffects from re-reading the argument and thus there is no point in making a private copy.

```cs
.method private hidebysig static
	string Name (
		class Customer customer
	) cil managed
{
	// Method begins at RVA 0x206b
	// Code size 12 (0xc)
	.maxstack 8

	IL_0000: ldarg.0          // read the argument
	IL_0001: brtrue.s IL_0005

	IL_0003: ldnull
	IL_0004: ret

	IL_0005: ldarg.0          // just read the argument again
	IL_0006: call instance string Customer::get_Name()
	IL_000b: ret
} // end of method Program::Name
```
