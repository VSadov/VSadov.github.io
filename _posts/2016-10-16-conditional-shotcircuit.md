---
title: Conditional member access operator (precedence and associativity).
tags: [CSharp, Conditional Access]
---
Conditional member access makes all the trailing member/element accesses and invocations applied conditionally as a whole. It might seem an intuitive behavior, but it caused a lot of controversy when the feature was designed.

A statement like

```cs
var lastName = customer?.Name.LastName;
```
translates into

```cs
var temp = customer;
lastName = temp != null ?       
                  //whole Name.LastName applied conditionally
                  temp.Name.LastName:
                  null;
```

NOT into

```cs
var temp = customer;
var name = temp != null ?       
                  //only Name applied conditionally
                  temp.Name:
                  null;

lastName = name.LastName:
```

Indeed, the second variant is not very useful. There is no point watsoever to null-check ```customer``` if the whole expression would result in throwing NullReferenceExeption regardless.

The first variant is how the cascading member accesses would be applied to a potentially null variable if written by hand.
So, if the overall semantic is intuitive, what was the reason for the contention?

In addition to the whole construct making sense, language designers also strive for  intuitive composition of the feature with other features and for a user accustomed to regular member accesses, conditional operator has a major quirk here.  

In a very loose sense the conditional member access can be seen as a short-circuiting operator with lower precedence than regular accesses while being _right-associative_ with respect to other trailing conditional accesses.  
(associativity and precedence are not exactly the right terms here, but I do not know what would fit better)

If we could use parenthesis to illustrate parts of chained member accesses applied conditionally, the example above would look like

```cs
// not a valid syntax, just using ( ) to show operand grouping
var lastName = customer?(.Name.LastName);
```

Theoretically, an alternative grouping could be:

```cs
// valid syntax, but not reflecting the actual semantics
var lastName = (customer?.Name).LastName;
```

The problem is that only the second variant is a valid syntax, but unlike if used with a chain of all-regular accesses, it changes the meaning of the expression in a way that negates the point of the conditional access. Similar problem would arise if ```customer?.Name``` was refactored into a temporary local. -  

The following refactoring of ```var lastName = customer?.Name.LastName;``` is not correct

```cs
var name = customer?.Name;
var lastName = name.LastName;
```

The refactored code would throw NullReferenceExeption regardless whether ```customer``` was ```null``` or not, assuming that ```Name``` is a reference type. If the type of ```Name``` is a struct, the refactored code would not even compile since ```name``` would have a nullable type.

The language designers were very concerned about "surprises" like above. It was very desirable to keep the "left-associativity" of the regular member accesses. Alternative mixed solutions were discussed:

- === Keep left-associativity, but make regular accesses behave like conditional accesses after "?." - essentially automatically turing trailing "." into "?.", "[]" into "?[]" and so on.
The problem with this approach is additional null check.

```cs
var lastName = customer?.Name.LastName;
```

would really behave like

```cs
var lastName = customer?.Name?.LastName;
```

The code will have to null-check results of ```Name```. If ```Name``` is not supposed to ever return ```null```, the check is redundant and in fact may conceal a bug if a bug results in ```Name``` returning ```null```.
This approach does not solve the "refactoring" issue. Extracting ```customer?.Name``` in a temp will still change the semantics.

- === Keep left-associativity, but give warning on trailing regular accesses, steering the user to the usage of conditional accesses.

It feels like this solution does not solve much at all, just makes the issues more apparent and shifts the eventual blame.


After several discussions it was concluded that the right-associative option is more useful and the apparent associativity distinction from regular member accesses is something that will not come up very often and users will just get accustomed to that with use.

The mental model for the execution of conditional accesses, chained with other, regular or conditional, member/element accesses or invocations is actually very simple. - The execution simply goes left-to-right and applies the operators to what we have so far. Every time the "?" is encountered, there is a null check on what we have. If the value happens to be ```null``` the rest of the expression is skipped and a ```null``` value with the type of the whole expression is produced as a result.

---  
Some more examples of how trailing accesses are grouped into conditional chains:

```cs
var result1 = customer?.Name.LastName?.Length.ToString();
// not a valid syntax, just using ( ) to show operand grouping
var result1 = customer?( .Name.LastName?( .Length.ToString() ) );

var result1 = customers?  [3].Orders?  [1].Destination.Address.Street;
// not a valid syntax, just using ( ) to show operand grouping
var result1 = customers?( [3].Orders?( [1].Destination.Address.Street ) );
```
