I will only summarize and explain the parts that I found especially useful or interesting. That means I have not covered all parts of his post. If you want an overview of all improvements, then I very much recommend that you read his full post: [Performance Improvements in .NET 8 by Stephen Toub - MSFT](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/)

### Dynamic PGO
I think the greatest improvement to .NET 8 is Dynamic PGO (Profile-Guided Optimization). The concept is not new to .NET 8, but .NET 8 is the first version where Dynamic PGO is turned on by default. So, what we get now in .NET 8 is actually the result of all the improvements and refinements made to Dynamic PGO since it was first introduced as a preview feature in .NET 6.

In broad terms, Dynamic PGO enables the JIT (Just-In-Time) compiler to observe which parts of the program are run the most while the program is running in production. It can then re-compile the parts that get run the most to make them more efficient when they are run in the future. You might ask, *"Why not compile all parts of the program to be the most efficient before it starts?"*. The answer to this question has multiple facets.

**Long startup time**: 
If the JIT compiler should try to compile all parts of the program before the program starts, then you might experience a very long startup time. If you have ever tried to run an AOT (Ahead Of Time) compilation of a program, then you know this can take a very long time. And the worst part is that it would use valuable time on compiling methods to be effective even though you might only run it once.

**Loss of insight**: 
When the program has already started, the JIT can get a greater insight into the actual values of certain fields and make smarter decisions. Stephen Toub gives an example of a `static readonly bool` which, once initialized, gets some value that could then be handled as a `const` when that code gets re-compiled, making the access to that faster while opening up for constant-inlining in places that it is used.

But how does it know what parts of the code get hit the most? They count how many times each method (and some other smaller parts) gets called. The JIT adds some instrumentation to all methods when it makes its first non-optimized compilation. This instrumentation is simply some code that increments by `1` every time that code is called. Then if this count hits some threshold for a method, the JIT will take the time to re-compile this to be more effective. A rather ingenious detail to this is that it also works when you work with multiple threads. Normally, when you try to increment the same number from multiple threads, we either get race conditions, meaning some of the increments will be lost, or we need to take a lock before reading and writing the number, which causes a huge performance penalty as the threads will have to wait for each other.

They solved this using randomization. They start by using the lock method, but then once the count surpasses `8192` (`2^13`), they switch to the non-locking method but only does the increment `50%` of the time decided by a random source but then increase it with `2` instead of `1` when it happens to minimize the chance of a race condition happening. Once it increases above `16384` (`2^14`) they do this once more, changing the probability of the increment happening to `25%` with the increment step being `4`. This continues for larger and larger numbers. This maintains a rather precise estimation of the actual count with minimal delay and performance penalty.

### GDV (Guarded Devirtualization)
The above counting trick is not only used to check which methods are called the most. It also checks which concrete types are used most often in places where virtual and interface types are used. The JIT then uses this information to generate a *fast path* for that concrete type. A fast path means that it adds a direct check for that type and then makes the invocation directly using that type if the type matches, and else makes the implementation lookup as normal. That's why it's called *Guarded* as we first guard for that common type. It does this for the single most used type, but it is possible to set the number of direct guards checks to another limit by setting the environment variable `DOTNET_JitGuardedDevirtualizationMaxTypeCheck`.

### Vectorization 
A trend over the last couple of .NET versions has been improvements to Vectorization. \* drom rolls \* and the trend continues!

Vectorization is when you utilize your hardware to do the same calculation to multiple numbers at once. .NET 7 added support for doing this for `128` and `256` bits at once. `256` bits are equivalent to `8` ints, so we could execute so-called SIMD (Single Instruction/Multiple Data) instructions on `8` ints, all in the same CPU cycle if the machine supports this. In .NET 8 they added support for vectorization of `512` bits, which means that we can make certain SIMD operations on `16` ints at once if the machine supports it. This means the execution time for simple arithmetic on large data arrays can be halved if the machine supports it. In .NET 7 these improvements were also utilized in the .NET standard `System.Linq` methods `Sum`, `Average`, `Min`, and `Max`. The `512` bit vectorization has likewise been added to these implementations in .NET 8 to make them even more efficient.

### Branching
We say that the program branches whenever a program has multiple paths depending on some condition. The CPU tries to predict which way the program will branch depending on what it thinks is most likely and then continues decoding the instructions on the most likely path while the branch outcome is being evaluated. This is called branch prediction. When branch prediction fails, it can be expensive. To get around this, we want to do branch-free code so that the branch prediction can never fail (as there will be none). The JIT can now emit conditional move instructions, making branch-free code much easier. Specifically, it now recognizes the pattern `(b ? 1 : 0)` where `b` is a boolean expression to be branchless. If we look at the following method:
```csharp
private int ModifyForRelevance(int score, bool isNew)
{
    if (isNew)
    {
        return score * 3;
    }
    else
    {
        return score * 2;
    }
}
```
This could be rewritten to the following to be branch-free in .NET 8:
```csharp
private int ModifyForRelevance(int score, bool isNew)
{
    return (isNew ? 1 : 0) * score * 3 + (isNew ? 0 : 1) * score * 2;
}
```
But that the code is branch-free doesn't mean it is more efficient. The first version of the method would generate a lot less code and would need fewer calculations, as we will always calculate both branches in the second version. So, for now, you probably shouldn't go out and use the `(b ? 1 : 0)` pattern in all parts of your code. This can be useful, though, when making methods that always take the same amount of time to execute. I don't think Stephen Toub touches on concrete use cases for this. But one such use case is to make constant-time implementations of cryptographic algorithms to prevent timing attacks.

### Bounds Checking
When accessing arrays, the JIT generates checks for the bounds of the arrays that throw the well-known `IndexOutOfRangeException` if you read outside the array by accident. But sometimes we know that the index we access can't possibly be outside the bounds. In .NET 8 the JIT understands more of these cases so that unnecessary checks are omitted from the emitted IL code. Stephen Toub gives multiple realistic examples of this:
```csharp
private int GetBucket(int[] buckets, int hashcode) => buckets[(uint)hashcode % buckets.Length];
```
The above indexes into an array using a large hash that might be larger than the actual size of the array. They use the modulus operator to get it within the bounds to get around this. This also means that it can never get outside the bounds, so the JIT doesn't need to generate the code that throws the `IndexOutOfRangeException`. Here is another example that has benefited from this:
```csharp
private bool IsQuoted(string s) => s.Length >= 2 && s[0] == '"' && s[^1] == '"';
```
The method first checks that the string is longer than `2` characters, as we can't have a quoted string shorter than `2` characters. In .NET 7 the JIT already recognized that the next sub-expression wouldn't need to make a bound check as it knew that the `0` index is safe to access when the array is `2` characters long. The new thing in .NET 8 is that it also recognizes that the last expression doesn't need bound checks as it knows that accessing the last element in the array is safe as there only needs to be at least `1` element in the array for this to be valid.

### Constant Folding
Constant folding is when the JIT can take an expression that only consists of constants and precompute the expression as it will always give the same result. .NET 8 has made many improvements to constant folding that make it able to recognize more types as being constant, and then employ the folding on these. From what I gathered from the post the most important feature is the support for constant folding of the lengths of `ReadOnlySpan`, `string`, and any type of array. This plays nicely together with the bounds-checking improvements as the JIT can now inline the length of a constant string as a constant itself which again means that it can access parts of the string within its bounds before even compiling, essentially compacting the whole expression to its result before emitting the IL code for that part. The following is a somewhat simple sample:
```
private static readonly string cat = "CAT";

private bool CatEndsWithT() => cat.Length == 3 && (cat[^1]  | 0x20) == 't';
```
The `CatEndsWithT` method would previously have needed to check for the length of the string `cat` both in the first sub-expression and when accessing the last element in the string. But now in .NET 8 the entire method can return `true` without looking at the string because of constant folding.

### Non-GC Heap
.NET 8 introduces a new heap, the Non-GC Heap. It is not something that most developers will use explicitly, but the JIT will use it in multiple scenarios. The Non-GC Heap is a heap parallel to the Garbage Collected heap, which the Garbage Collector will not manage. It is intended for memory that is not going to change throughout the lifetime of the application. .NET uses this to store constants and literals so that the JIT can point to them directly in the Non-GC Heap instead of first having to find out where it is in the normal heap as that can be re-arranged when the Garbage Collector cleans up memory. This means that the following things can now be referenced directly, which saves one move instruction:
- String Literals
- `typeof(T)` for types that are not dynamically loaded.
- `Array.Empty<T>()` again for types not dynamically loaded.
- Value types in static fields (which were previously boxed).

### Closing notes
Now, we have seen the improvements I found most insightful from Stephen Toub's post. And I restate that you should read his full post to get all the details. There were, of course, parts that I didn't cover in this post. Many of the areas I didn't cover focus on improvements to different kinds of arrays/collections and common actions involving these. I would especially like to use spans more, as they bring many benefits. And obviously, I'm also very interested in the improvements that influence Blazor WASM. But I think I will cover that in another post. 
