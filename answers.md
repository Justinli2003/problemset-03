# CMPS 6610 Problem Set 03
## Answers

**Name:**_________________________


Place all written answers from `problemset-03.md` here for easier grading.




- **1b.**

  `isearch` uses `iterate` with a boolean accumulator ("have I seen `x` yet?"),
  OR-ing in `(y == x)` at each step. Each step does `O(1)` work, and there are
  `n` steps:

  - Work: `W(n) = W(n-1) + O(1)` ==> **`O(n)`**
  - Span: `S(n) = S(n-1) + O(1)` ==> **`O(n)`**

  The span equals the work because `iterate` is inherently sequential: step
  `i+1` cannot begin until step `i` has produced the accumulator it consumes.
  There is a single chain of `n` dependent operations, so the parallelism is
  `W/S = O(1)` and extra processors buy us nothing.

  (Aside: the `iterate` given to us slices `a[1:]` on every call, which copies
  the rest of the list and makes the real Python cost `O(n^2)`. In the cost
  model we assume the step itself is `O(1)`.)



- **1d.**

  `rsearch` first maps `L` into the list of booleans `(y == x)`, then reduces
  that list with logical OR (associative, with identity `False`). The map is
  needed because `reduce` returns `a[0]` directly when `|a| == 1` — it never
  applies `f` to the identity — so the elements have to already live in the
  answer domain.

  - the map: `O(n)` work, `O(1)` span (every element is independent)
  - the reduce: `reduce` splits into two halves that recurse in parallel and
    combines with one `O(1)` call to `f`:
    - Work: `W(n) = 2W(n/2) + O(1)` ==> **`O(n)`** (leaf-dominated, since
      `n^(log_2 2) = n` dominates the `O(1)` combine)
    - Span: `S(n) = S(n/2) + O(1)` ==> **`O(log n)`**

  Total: **work `O(n)`, span `O(log n)`.**

  So `rsearch` does the same asymptotic work as `isearch` but has exponentially
  better span. The reason is that OR is associative, so the `n` combines can be
  arranged as a balanced binary tree of depth `log n` instead of a chain of
  length `n`. The parallelism is `O(n / log n)`.



- **1e.**

  `ureduce` splits the input at the one-third point instead of the midpoint, so
  the two recursive calls get `n/3` and `2n/3` elements. The output is identical
  (OR is associative, so any parenthesization gives the same answer), but the
  recursion tree is now unbalanced:

  - Work: `W(n) = W(n/3) + W(2n/3) + O(1)` ==> **`O(n)`**

    The two subproblem sizes still sum to `n`, so the tree has `n` leaves and
    `O(n)` internal nodes total, each doing `O(1)` work.

  - Span: `S(n) = max(S(n/3), S(2n/3)) + O(1) = S(2n/3) + O(1)` ==>
    **`O(log n)`**

    The critical path always follows the larger `2n/3` branch, so its length is
    `log_{3/2} n` rather than `log_2 n`.

  Total: **work `O(n)`, span `O(log n)` — asymptotically the same as `reduce`.**
  The only thing that changes is the constant: `log_{3/2} n = log_2 n / log_2(3/2)`
  is about `1.71x` deeper, so `ureduce` has a longer critical path by a constant
  factor. An uneven split hurts only when it is uneven enough to be
  *asymptotically* unbalanced (e.g. splitting off one element at a time, which
  degenerates into `iterate` with span `O(n)`); a constant-fraction split like
  1/3–2/3 still gives logarithmic depth.

  (Note: as written, `ureduce` actually calls `reduce` for its subproblems
  rather than recursing into itself. That makes the tree unbalanced only at the
  top level and balanced below, which still gives `O(n)` work and `O(log n)`
  span. The analysis above is for the intended fully-recursive version.)



- **2a.**

  **Idea.** The sequential way to dedup is to sweep left to right keeping a set
  of elements seen so far — but that is an `iterate`, with span `O(n)` and no
  parallelism. To parallelize, we replace the *global* question "has this value
  appeared before?" with a *local* one: if we sort the elements by
  `(value, original index)`, then all copies of a value sit next to each other
  in increasing index order, so the first occurrence of each value is exactly an
  element whose left neighbor differs from it. That test is local, so a `map`
  and a `filter` answer it for every position in parallel. Sorting by the
  original index at the end restores the input order.

  **SPARC specification.**

  ```
  dedup (A) =
    let
      (* 1. tag each element with its position *)
      B = < (A[i], i) : 0 <= i < |A| >

      (* 2. sort lexicographically: by value first, ties broken by index *)
      C = sort ((a,i),(b,j) => a < b or (a = b and i < j)) B

      (* 3. keep only the first copy of each value, i.e. the elements whose
            left neighbour holds a different value *)
      D = < C[k] : 0 <= k < |C| | k = 0 or val(C[k]) != val(C[k-1]) >

      (* 4. put the survivors back into their original relative order *)
      E = sort ((a,i),(b,j) => i < j) D
    in
      < val(e) : e in E >
    end
  ```

  **Correctness.** Step 2 groups equal values into contiguous runs, ordered
  within a run by original index. Step 3 keeps exactly one element per run — the
  one with the smallest index, i.e. the earliest occurrence. So `D` contains
  each distinct value exactly once, represented by its first occurrence. Step 4
  sorts those representatives by index, which is precisely the order they
  appeared in `A`.

  **Work and span.**

  | Step | Work | Span |
  |------|------|------|
  | 1. tag (`map`) | `O(n)` | `O(1)` |
  | 2. `sort` (parallel mergesort) | `O(n log n)` | `O(log^2 n)` |
  | 3. `filter` on adjacent pairs | `O(n)` | `O(log n)` |
  | 4. `sort` by index | `O(n log n)` | `O(log^2 n)` |
  | 5. strip tags (`map`) | `O(n)` | `O(1)` |

  - **Work: `O(n log n)`**, dominated by the two sorts.
  - **Span: `O(log^2 n)`**, also dominated by the sorts.

  (The `filter` is `O(log n)` span rather than `O(1)` because it needs a `scan`
  to compute where each surviving element goes in the output.)

  Parallelism is `O(n / log n)`, versus `O(1)` for the sequential
  set-based sweep. If we did not care about preserving order, we could drop
  step 4 and the index tags entirely, which would halve the constant but not
  change the asymptotics.



- **2b.**

  Now we have `m+1` lists of `n` elements each, so `N = (m+1)n` elements total,
  and order no longer matters. Dropping the ordering requirement means we can
  skip the index bookkeeping and the second sort of part a).

  **SPARC specification.**

  ```
  multi-dedup (A) =
    let
      (* 1. dedup each list locally, in parallel; keep results sorted *)
      D = < local-dedup (A[i]) : 0 <= i <= m >

      (* 2. combine all the local results with a balanced tree of merges *)
      F = reduce merge-dedup <> D
    in
      F
    end

  local-dedup (X) =
    let S = sort X
    in  < S[k] : 0 <= k < |S| | k = 0 or S[k] != S[k-1] >  end

  merge-dedup (X, Y) =
    let Z = merge X Y        (* merge two already-sorted sequences *)
    in  < Z[k] : 0 <= k < |Z| | k = 0 or Z[k] != Z[k-1] >  end
  ```

  `merge-dedup` is associative with identity `<>` (it amounts to set union on
  sorted sequences), which is exactly what `reduce` requires in order to build a
  balanced combining tree.

  **Work and span.**

  - Step 1: each `local-dedup` costs `O(n log n)` work and `O(log^2 n)` span.
    There are `m+1` of them and they are independent, so together they cost
    `O(mn log n)` work and `O(log^2 n)` span.
  - Step 2: the `reduce` tree has `log(m+1)` levels. Each level merges a total
    of at most `N` elements, at `O(1)` work per element, so each level costs
    `O(N)` and the whole tree costs `O(N log m)` work. A parallel merge of two
    sorted sequences has span `O(log^2 N)`, so the tree's span is
    `O(log m * log^2 N)`.

  - **Work: `O(N log N)`**, i.e. `O(mn log(mn))`.
  - **Span: `O(log m * log^2 N)`**, which is polylogarithmic in the total input
    size.

  **Comparison with part a).** The work grows by roughly a factor of `m+1`,
  which is unavoidable — every one of the `N` elements has to be looked at at
  least once, so `Omega(N)` work is a lower bound. What is striking is the span:
  going from `n` elements to `(m+1)n` elements raises the span only from
  `O(log^2 n)` to `O(log^2 N)` — polylogarithmic in both cases. In other words,
  with enough processors, deduplicating a thousand lists takes barely longer
  than deduplicating one, because all the per-list work happens simultaneously
  and the results are combined in a tree of depth `log m` rather than a chain of
  length `m`. The parallelism is `O(N log N / (log m log^2 N))`, which grows
  nearly linearly in `N`.

  A simpler alternative is to `flatten` all the lists into one sequence of `N`
  elements and run part a)'s algorithm on it: that gives `O(N log N)` work and
  `O(log^2 N)` span, which is actually slightly *better* span. But in a real
  distributed setting the version above is preferable, because deduplicating
  locally first shrinks each list before anything crosses the network — the
  asymptotic work is the same, but far less data gets moved between machines,
  and network traffic is the real bottleneck in the cloud setting.



- **2c.**

  Yes — several of them are essential, and one notable one is useless.

  **Useful:**

  - **`map`** — tagging every element with its index, and testing whether an
    element differs from its left neighbor. Both are per-element, independent
    computations: `O(n)` work and `O(1)` span. This is what turns the "first
    occurrence" test into a parallel operation.
  - **`filter`** — keeping only the elements that begin a run of equal values.
    This is the step that actually removes the duplicates.
  - **`scan`** — used inside `filter` to compute each surviving element's
    position in the output array (a prefix sum over the 0/1 "keep" flags).
    Without a scan we would not know where to write each result in parallel.
  - **`reduce`** — the key operation for part b). Because `merge-dedup` is
    associative with identity `<>`, `reduce` can combine the `m+1` per-list
    results with a balanced tree of depth `log m` instead of folding them one
    at a time, turning an `O(m)` critical path into an `O(log m)` one.
  - **`flatten`** — concatenating the collection of lists in part b).

  **Not useful:**

  - **`iterate`** — the natural sequential algorithm ("walk the list carrying a
    set of everything seen so far") is exactly an `iterate`, and `iterate` has
    span `O(n)`: each step depends on the accumulator produced by the previous
    one, so there is no parallelism at all. This is precisely why both
    algorithms above go through sorting instead. Sorting is the trick that
    converts a question about *all preceding elements* (inherently sequential)
    into a question about *the immediately preceding element* (perfectly
    parallel), at a cost of a `log n` factor in work.



- **3b.**

  `parens_match_iterative` calls `iterate` once over the `n` input characters.
  `parens_update` does a constant amount of work per character (compare, then
  increment/decrement a counter or latch it to `None`).

  - Work: `W(n) = W(n-1) + O(1)` ==> **`O(n)`**
  - Span: `S(n) = S(n-1) + O(1)` ==> **`O(n)`**

  Again the span equals the work: the counter for character `i+1` cannot be
  computed until the counter for character `i` is known, so the whole
  computation is one dependency chain of length `n` and the parallelism is
  `O(1)`. The work is optimal — you must read every character — but there is no
  parallel speedup available.



- **3d.**

  `parens_match_scan` makes one `map`, one `scan`, and one `reduce` pass:

  - **`map`** of `paren_map` over the input: every character is handled
    independently, so `O(n)` work and `O(1)` span.
  - **`scan`** with `plus` (the efficient contraction-based version). Contraction
    pairs up adjacent elements, recurses on the `n/2` pairwise sums, then expands
    the result back out; the contract and expand steps are parallel `map`s:
    - Work: `W(n) = W(n/2) + O(n)` ==> **`O(n)`** (root-dominated, since the
      per-level cost halves geometrically)
    - Span: `S(n) = S(n/2) + O(1)` ==> **`O(log n)`**
  - **`reduce`** with `min_f` over the prefix sums, to check that no prefix ever
    goes negative:
    - Work: `W(n) = 2W(n/2) + O(1)` ==> **`O(n)`**
    - Span: `S(n) = S(n/2) + O(1)` ==> **`O(log n)`**

  Total: **work `O(n)`, span `O(log n)`.** Same work as the iterative solution,
  but with `O(n / log n)` parallelism instead of none. The reason this works is
  that the running counter, which looked inherently sequential in 3b, is just a
  prefix sum — and addition is associative, so the prefixes can all be computed
  in a tree of depth `log n`.



- **3f.**

  `parens_match_dc_helper` splits the list in half, solves the two halves
  recursively in parallel, and merges the two `(R, L)` pairs in constant time
  with `m = min(j, k)`, returning `(i + k - m, l + j - m)`.

  - Work: `W(n) = 2W(n/2) + O(1)` ==> **`O(n)`**

    Leaf-dominated: `n^(log_2 2) = n` dominates the `O(1)` combine, so the cost
    is proportional to the `n` leaves.

  - Span: `S(n) = S(n/2) + O(1)` ==> **`O(log n)`**

    Both recursive calls run in parallel, so only one of them lies on the
    critical path, and the merge adds `O(1)`.

  Total: **work `O(n)`, span `O(log n)`.**

  **Summary of the three solutions:**

  | Solution | Work | Span | Parallelism |
  |----------|------|------|-------------|
  | 3a. `iterate` | `O(n)` | `O(n)` | `O(1)` |
  | 3c. `scan` | `O(n)` | `O(log n)` | `O(n / log n)` |
  | 3e. divide & conquer | `O(n)` | `O(log n)` | `O(n / log n)` |

  All three do optimal `O(n)` work. The scan and divide-and-conquer versions
  match each other asymptotically, but the divide-and-conquer version has the
  smallest constants — it makes a single pass over the recursion tree carrying
  two integers, whereas the scan version builds an intermediate list of prefix
  sums and then makes a separate reduce pass over it. In practice I would use
  3e.

  (Note: in the given Python code the slicing `mylist[:mid]` / `mylist[mid:]`
  copies the sublists, which adds `O(n)` per level and makes the real work
  `O(n log n)`. In the cost model we assume sequences can be split in `O(1)`.)
