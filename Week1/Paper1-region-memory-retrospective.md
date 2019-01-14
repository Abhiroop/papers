## Title : A Retrospective on Region-Based Memory Management

## [Link](http://citeseerx.ist.psu.edu/viewdoc/download;jsessionid=92F4C364284417DD25BF96CEABB69D18?doi=10.1.1.64.160&rep=rep1&type=pdf)

Summary: This paper looks back at the origin and development of the region based memory management discipline. The fundamental inspiration of Region based memory management is the Algol stack discipline. Stack discipline can provide better cache locality and lesser fragmentation than heap allocation(Although theoretically heap allocation is more efficient than stack allocation [1]). The stack discipline essentially stores the memory proportional to the depth of the call stack unlike heap allocation which could theoretically store the entire call tree.

A region based memory model consists of a stack of regions. Everything including function closures are put into regions. All well typed expressions are transformed into region annotated expressions:
1. `e1 at p` -> Whenever an expression produces a value like constant expressions, the value at e1 was placed at the region bound to the region variable `p`
2. `letregion p at e2` -> This introduces a region variable `p` with local scope e2. At runtime, first an unused region, `r`, was allocated and bound to p. Then e2 was evaluated, probably using r or other regions on the stack. Finally, r was deallocated. The letregion expression was the only way of introducing and eliminating regions. Hence regions were allocated and de-allocated in a stack-like manner. A set of region inference rules were also introduced.


The results were terrible owing to:
1. When a function f , say, returned a result in a region, then all calls of f had to return their result in the same region. Thus the region had to be kept alive until no result of f was needed (which was very conservative).
2. In particular, when a function called itself recursively, the result of the recursive call had to be put in the same region as the result of the function (even in cases where the recursive call produced a result that was not part of the result of the function).

Solution 1: Region polymorphic functions were introduced, which accepted region parameters. 2 new annotations were introduced as a result: `letregion f[p1,...pk](x) = e1 in e2` and the other `f[p1,....pk]`


Solution 2: Polymorphic recursion

Translates

```SML
letrec fac[r1](n) =
      if n = 0 then 1 at r1
      else (n * fac[r1](n-1)) at r1
in fac[r0] 100
```

to

```SML
letrec fac[r1](n)=
        if n = 0 then 1 at r1
        else letregion r2
             in (n * fac[r2](n-1)) at r1
             end
    in fac[r0] 100
```

All the values pile up in the same region in the first case but a new region is spawned for every recursion call in the second case.

Followed by that `storage mode analysis` was developed which allowed the compiler to overwrite/reset a region prior to allocation if it concluded that there are no live values in the region.


### MLKit
---------
In MLKit the runtime system represented regions by linked lists of fixed size region pages because the size of each region was not known and could potentially be unknown. Experiments revealed that many regions only ever contained one value. Such regions were placed on the stack rather than allocating region pages for them.

`Multiplicity inference analysis` was developed which finds out **upper bound on the number of values written into a region.** Experiments showed that region size was generally one or infinite(bound could not be determinded). As a result the finite regions were part of the activation record while an infinite region was a linked list of fixed size region pages, allocated from a free list of region pages.


For proving the correctness of region inference algorithms two different algos evolved:

1. syntax directed and based on algorithm W and fixed point iterating for dealing with polymorphic recursion.
2. constraint based

Second was faster and more space consuming. Hence the first one was chose.


Followed by that a region profiler was developed.


### Modules and separate compilation
------------------------------------

A scheme based on the `static interpretation of modules`, where the module language was regarded as a linking language, was developed. Type checking happenned but the code generation was delayed until the application of the functor. Delaying code generation until functor application time was not feasible for large programs unless it was integrated with a mechanism for avoiding unneccesary recompilation of program units and functor bodies upon change of source code. To solve this problem, the static interpretation of modules scheme collected information, which for each program unit or functor body told which other units it depended on. Such information included region type schemes for free identifiers of the program unit. Upon modification of a program unit, the scheme used the collected information to determine, for each program unit and each
functor application, if recompilation was necessary.

### GC and Regions
------------------

The idea was to perform a Cheney copying collection of all regions on the region stack but to do it in such a way that two live values in the same region remained in the same region even after the copying GC finished. The region typing rules automatically prevents dangling pointers.

Experiments showed that what strategy to use (i.e., region inference alone, garbage collection alone, or a combination of the two) is not a clear cut and depends on the program. However, the combination of region inference and garbage collection did give the programmer the flexibility to either optimise for regions or choose not to and instead use the garbage collector as a fall-back opportunity.


Suggested future work involves: Investigating the use of infinite regions by using a single infinite region and using a more sophisticated (generational collector). Instead of garbage collecting at a function point we should collect at each allocation point. Another possibility is to explicitly use region annotations, something which the Cyclone project is experimenting with.




 
[1] Garbage collection can be faster than stack allocation. - Andrew Appel. [Link](https://goo.gl/vnTvdq)

