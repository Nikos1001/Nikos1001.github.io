
## Fighting memory corruption with memory corruption

Here's a fun low-level hack I came up with while hacking on an interpreter for my programming language Nost: say you're using memory arenas in order to speed up allocations, increase cache utilization, and most importantly, reduce the anxiety around remembering to call a `free()` for every `malloc()`. In C, here's how an implementation of a simple arena might look like:

```c

typedef struct {
    void* base;
    void* curr;
} arena;

arena makeArena() {
    arena res;
    res.base = res.curr = malloc(16 * 1024); // 16K is plenty!
    return res;
}

void* arenaAlloc(arena* arn, size_t size) {
    void* res = arn->curr;
    arn->curr += size;
    return res;
}

void arenaClear(arena* arn) {
    arn->curr = arn->base;
}

int main() {
    arena arnold = makeArena();

    int* importantNumber = arenaAlloc(&arnold, sizeof(int));
    *importantNumber = 33;

    int* anotherImportantNumber = arenaAlloc(&arnold, sizeof(int));
    *anotherImportantNumber = 99;

    printf("%d\n", *importantNumer); // prints 33

    arenaClear(&arnold);
}

```

A nice and simple allocator! Let's visualize what it's doing. The arena holds a 16K buffer of memory that we allocated with malloc, which is really just a massive array of bytes. Initially, all the bytes are random values from wherever we got that buffer. The arena holds two pointers into that buffer: `base`, which always points to the start of the buffer, and `curr` for the place where the next allocation will occur. Here's what the arena looks like initially:

```
 base
 curr
  V
| ?? | ?? | ?? | ?? | ?? | ?? | ?? | ?? | ?? | ??
```

"Allocating" memory with an arena is very simple: just return the `curr` pointer, and advance it by the size of the allocation. So after allocating and assigning `importantNumber`,

```
 base                curr
  V                   V
| 0  | 0  | 0  | 21 | ?? | ?? | ?? | ?? | ?? | ??
  ^
  importantNumber
```

The bytes are displayed in hex for asthetic purposes, so 33 becomes 0x21. Then we allocate `anotherImportantNumber`,

```
 base                                    curr
  V                                       V
| 0  | 0  | 0  | 21 | 0  | 0  | 0  | 63 | ?? | ??
  ^                   ^
  importantNumber     anotherImportantNumber
```

Then, to "free" the memory we used up, we just need to move the `curr` pointer to the `base` pointer to make space for future allocations.

```
 base
 curr                                    
  V                                     
| 0  | 0  | 0  | 21 | 0  | 0  | 0  | 63 | ?? | ??
  ^                   ^
  importantNumber     anotherImportantNumber
```

But uh oh, `importantNumber` and `anotherImportantNumber` still exist... at the moment, reading from them would return the values they pointed to originally, but what if we tried to allocate an `int` onto the arena now? Say we do something like,

```
int* mostImportantNumber = arenaAlloc(&arnold, sizeof(int));
*mostImportantNumber = 42
```

The arena would then read:

```
 base                curr                                
  V                   V                                     
| 0  | 0  | 0  | 2A | 0  | 0  | 0  | 63 | ?? | ??
  ^                   ^
  importantNumber     anotherImportantNumber
  mostImportantNumber
  
```

Now if we read from `importantNumber`, we'll actually be reading memory intended for `mostImportantNumber`! At best, this will somehow lead to a crash during development that the programmer can then dutifuly hunt down and bring to justice. It will be a pain to find, though. But worst of all, these kinds of bugs can easily slip through and go unnoticed until they creep up somewhere else and cause all kinds of nightmares, at which point the unfortunate developer will pay the ultimate price. So the question is: how do you make sure that you find all the bugs of this sort during development to avoid provoking the wrath of the powers that be?

Here's the solution I came up with while implementing Nost: *intentional memory corruption*. It works like so: when you clear the arena, set every byte allocated to a random value. In code,

```c
void arenaClear(arena* arn) {
    for(int i = 0; i < 16 * 1024; i++)
        ((char*)arn->base)[i] = rand();
    arn->curr = arn->base;
}
```

Now, let's go back in time right before we clear our arena. 

```
 base                                    curr
  V                                       V
| 0  | 0  | 0  | 21 | 0  | 0  | 0  | 63 | ?? | ??
  ^                   ^
  importantNumber     anotherImportantNumber
```

At this point, reading from `importantNumber` still yields a reasonable 33. Now let's clear the arena:

```
 base
 curr                                    
  V                                     
| 10 | DE | 77 | 53 | C5 | 4E | 6F | FD | 56 | 0F
  ^                   ^
  importantNumber     anotherImportantNumber
```

Unlike `rand()`, the numbers shown above are truly random - well, as random as you can reasonably get without connections to quantum physists - since they were generated using [random.org](https://www.random.org/). But the effect is the same: now as soon as you clear the arena, `importantNumber` goes from pointing to 33 to now pointing to 283014995. If you then go on to use `importantNumber` for something later in the code, it will be much more likely to crash or to produce obviously garbled results. This, of course, is not a technique guaranteed to find bugs 100% of the time - this is a hacky, heuristic approach that can help reveal these sorts of bugs more often. But it is a genuinely useful hack - so far, it helped me discover one GC-related bug in my interpreter, which is pretty significant given that programming language implementations are required to be absolutely bullet-proof. Personally, I like this trick just because of how absolutely hilarious it is: using intentional memory corruption to stifle unintentional memory corruption! 

If you're using ASAN(you should be - it's awesome!) you can make a more boring yet probably more effective version of this trick. Every time you clear the arena, free the old buffer and replace it with a new one. Assuming that malloc doesn't place the new buffer in the exact place of the old one, reading from invalidated pointers will be flagged as a use-after-free error by ASAN. 

Obviously, using this comes at a performance cost whenever you clear an arena, so this should be enabled with conditional compilation during development. I find that adding runtime checks and error-detectors of this nature helps a lot with tracking down safety problems and vulnerabilities. [A hard drill makes an easy battle](https://www.joelonsoftware.com/2001/11/20/a-hard-drill-makes-an-easy-battle/).