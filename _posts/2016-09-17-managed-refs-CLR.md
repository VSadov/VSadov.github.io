---
title: Managed pointers.
tags: [CLR, refs]
---
After untangling some confusion about [ref variables in C#]({% post_url 2016-09-05-refs-not-ptrs %}), lets bring some confusion back.  
Many questions in [that post]({% post_url 2016-09-05-refs-not-ptrs %}), while not applicable to C#, are meaningful when asked about CLR managed pointers.

Firstly, C# has a concept of unmanaged pointers. Unmanaged pointers are just types with certain operations like indirection (*), member access (->), indexing ([]), etc...
Compared to managed pointers, unmanaged pointers are rather boring. They are basically just strongly typed pointer-sized integers. Both ```int*``` and ```long*``` are simply two numbers and in fact could have same numerical value. The only difference is the granularity of indexing and interpretation of dereferencing operations.

Managed pointers are much more "magical". Similarly to object references ```O```, managed pointers ```&``` can point to managed heap objects. The main difference is that ```O``` always points to a whole object, while ```&``` points to variables that are parts of something else local slots or method parameters, array elements, fields, including fields of objects on the managed heap.
Since ```&``` can point to managed memory, they are reported to GC. They are counted as roots for reachability purposes and will be adjusted by GC during compaction. I.E. a live managed pointer to a field will keep the whole object alive and when GC needs to move the object in memory, the pointer will be adjusted.  
It is permitted for ```&``` to refer to objects not managed by GC - like locals, parameters or native memory. They are all still reported to GC. It is up to GC to sort them out.

The GC "magic" results in certain limitations:  

* Addition, subtraction and incrementing operations are mostly meaningless for managed pointers. When the referent can move at any time, such operations are hard to define. In rare cases subtraction can be used to get the byte distance between ```&```s to elements of the same array and that only works because the distance is not changed by GC and subtraction itself is an atomic operation so GC cannot happen while subtraction is in progress.    
In fact the aliasing "true pointer" implementation is not strictly guaranteed. For example in calls done via remoting it is permitted for a conforming CLI implementation to implement ```&``` parameters as copy-in/copy-out.

* Fields and array elements are not permitted to have ```&``` types. ```&``` cannot be boxed either.
These restrictions are a bit artificial. It just makes the job of GC easier if ```&``` themselves are never on the heap. In theory ```&``` fields and the like could be permitted and would allow for strange object topologies that are entirely alive via referring to each other via internal ```&``` pointers. There is, however, not a lot of value in that.

* Managed pointer cannot point to another managed pointer. Since managed pointers cannot be on the heap, there is a little reason to allow this and disallowing makes the type system simpler. Note that it is permitted for a ```&``` to point to a regular reference type variable ```O```.

* Managed pointers ```&``` are not interchangeable with object references ```O``` - they never point to same locations. While it is possible to cast ```&``` to an unmanaged pointer ```*```, it is extremely unsafe. If the referent was an unpinned heap object, using such pointer can cause AVs or heap corruptions.

* Since ```&``` can point to stack locals and parameters, a great care is required to ensure that pointers to not outlive the referents to preserve type safety. Indirect read via a dangled pointer can result in managed references pointing to random locations.  

Considering the "magic" nature of managed pointers and corresponding limitations, not a lot of useful scenarios would be enabled by exposing them directly. Besides, having multiple incompatible kinds of pointers side-by-side could be unnecessary confusing. Those are the main reasons for C# to have features like ref parameters, locals and returns, which are just a thin veneer above managed pointers and yet typesafe, while not exposing managed pointers themselves as a language feature.

Yes, a managed pointer can be ```null```, what else would it be before assigning anything? However in a well-formed C# program with no unsafe/reflection tricks or interop, you should never see a nullptr ```ref``` variable.
