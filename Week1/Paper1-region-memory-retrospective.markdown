##Title : A Retrospective on Region-Based Memory Management

##Link : http://citeseerx.ist.psu.edu/viewdoc/download;jsessionid=92F4C364284417DD25BF96CEABB69D18?doi=10.1.1.64.160&rep=rep1&type=pdf

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








 
[1] Garbage collection can be faster than stack allocation. - Andrew Appel. Link: ftp://ftp.cs.princeton.edu/reports/1986/045.pdf

