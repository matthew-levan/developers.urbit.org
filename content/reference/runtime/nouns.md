+++
title = "Land of Nouns"
weight = 3
+++
The division between `c3` and `u3` is that you could theoretically
imagine using `c3` as just a generic C environment.  Anything to do
with nouns is in `u3`.

### u3: a map of the system

These are the symbols you'll need to know about to program in `u3`.
All files listed below are found in the 
[`pkg/noun`](https://github.com/urbit/vere/tree/develop/pkg/noun) 
directory. Symbols follow this pattern:

```
prefix    purpose                      .h            .c
-------------------------------------------------------------------
u3a_      allocation                   allocate.h    allocate.c
u3e_      persistence                  events.h      events.c
u3h_      hashtables                   hashtable.h   hashtable.c
u3i_      noun construction            imprison.h    imprison.c
u3j_      jet control                  jets.h        jets.c
u3l_      logging                      log.h         log.c
u3m_      system management            manage.h      manage.c
u3n_      nock computation             nock.h        nock.c
u3o_      command-line options         options.h     options.c
u3r_      noun access (error returns)  retrieve.h    retrieve.c
u3s_      noun serialization           serial.h      serial.c
u3t_      profiling                    trace.h       trace.c
u3u_      urth (memory management)     urth.h        urth.c
u3v_      arvo                         vortex.h      vortex.c
u3x_      noun access (error crashes)  xtract.h      xtract.c
u3z_      memoization                  zave.h        zave.c
u3k[a-g]  jets (transfer, C args)      jets/k.h      jets/[a-g]/*.c
u3q[a-g]  jets (retain, C args)        jets/q.h      jets/[a-g]/*.c
u3w[a-g]  jets (retain, nock core)     jets/w.h      jets/[a-g]/*.c
```

Additionally, various noun type definition are found in `pkg/noun/types.h`.

### u3: noun internals

A noun is a `u3_noun` - currently defined as a 32-bit `c3_w`.
(This is zero-indexed so bit `31` is the high bit.)

If your `u3_noun` is less than `(1 << 31)`, it's a direct atom.
Every unsigned integer between `0` and `0x7fffffff` inclusive is
its own noun.

If bit `31` is set in a `u3_noun` and bit `30` is `1` the noun
is an indirect cell.  If bit `31` is set and bit `30` is `0` the
noun is an indirect atom.  Bits `29` through `0` are a word
pointer into the loom - see below.  The structures are:

```c
typedef struct {
  c3_w mug_w;
  c3_w len_w;
  c3_w buf_w[0];    //  actually [len_w]
} u3a_atom;

typedef struct {
  c3_w    mug_w;
  u3_noun hed;
  u3_noun tel;
} u3a_cell;
```

The only thing that should be mysterious here is `mug_w`, which
is a 31-bit lazily computed nonzero short hash (FNV currently,
soon Murmur3).  If `mug_w` is 0, the hash is not yet computed.
We also hijack this field for various hacks, such as saving the
new address of a noun when copying over.

Also, the value `0xffffffff` is `u3_none`, which is never a valid
noun.  Use the type `u3_weak` to express that a noun variable may
be `u3_none`.

### u3: reference counts

The only really essential thing you need to know about `u3` is
how to handle reference counts.  Everything else, you can skip
and just get to work.

u3 deals with reference-counted, immutable, acyclic nouns.
Unfortunately, we are not Apple and can't build reference
counting into your C compiler, so you need to count by hand.

Every allocated noun (or any allocation object, because our
allocator is general-purpose) contains a counter which counts the
number of references to it - typically variables with type
`u3_noun`.  When this counter goes to 0, the noun is freed.

To tell `u3` that you've added a reference to a noun, call the
function `u3a_gain()` or its shorthand `u3k()`.  (For your
convenience, this function returns its argument.)  To tell `u3`
that you've destroyed a reference, call `u3a_lose()` or `u3z()`.

(If you screw up by decrementing the counter too much, `u3` will
dump core in horrible ways.  If you screw up by incrementing it
too much, `u3` will leak memory.  To check for memory leaks,
set the `bug_o` flag in `u3e_boot()` - eg, run `vere` with `-g`.
Memory leaks are difficult to debug - the best way to handle
leaks is just to revert to a version that didn't have them, and
look over your code again.)

(You can gain or lose a direct atom.  It does nothing.)

### u3: reference protocols

**THIS IS THE MOST CRITICAL SECTION IN THE `u3` DOCUMENTATION.**

The key question when calling a C function in a refcounted world
is what the function will do to the noun refcounts - and, if the
function returns a noun, what it does to the return.

There are two semantic patterns, `transfer` and `retain`.  In
`transfer` semantics, the caller "gives" a use count to the
callee, which "gives back" any return.  For instance, if I have

```c
    {
      u3_noun foo = u3i_string("foobar");
      u3_noun bar;

      bar = u3f_futz(foo);
      [...]
      u3z(bar);
    }
```

Suppose `u3f_futz()` has `transfer` semantics.  At `[...]`, my
code holds one reference to `bar` and zero references to `foo` -
which has been freed, unless it's part of `bar`.  My code now
owns `bar` and gets to work with it until it's done, at which
point a `u3z()` is required.

On the other hand, if `u3f_futz()` has `retain` semantics, we
need to write

```c
    {
      u3_noun foo = u3i_string("foobar");
      u3_noun bar;

      bar = u3f_futz(foo);
      [...]
      u3z(foo);
    }
```

because calling `u3f_futz()` does not release our ownership of
`foo`, which we have to free ourselves.

But if we free `bar`, we are making a great mistake, because our
reference to it is not in any way registered in the memory
manager (which cannot track references in C variables, of
course).  It is normal and healthy to have these uncounted
C references, but they must be treated with care.

The bottom line is that it's essential for the caller to  know
the refcount semantics of any function which takes or returns a
noun.  (In some unusual circumstances, different arguments or
returns in one function may be handled differently.)

Broadly speaking, as a design question, retain semantics are more
appropriate for functions which inspect or query nouns.  For
instance, `u3h()` (which takes the head of a noun) retains, so
that we can traverse a noun tree without constantly incrementing
and decrementing.

Transfer semantics are more appropriate for functions which make
nouns, which is obviously what most functions do.

In general, though, in most places it's not worth thinking about
what your function does.  There is a convention for it, which
depends on where it is, not what it does.  Follow the convention.

### u3: reference conventions

The `u3` convention is that, unless otherwise specified, **all
functions have transfer semantics** - with the exception of the
prefixes: `u3r`, `u3x`, `u3z`, `u3q` and `u3w`.  Also, within
jet directories `a` through `f` (but not `g`), internal functions
retain (for historical reasons).

If functions outside this set have retain semantics, they need to
be commented, both in the `.h` and `.c` file, with `RETAIN` in
all caps.  Yes, it's this important.

### u3: system architecture

If you just want to tinker with some existing code, it might be
enough to understand the above.  If not, it's probably worth
taking the time to look at `u3` as a whole.

`u3` is designed to work as a persistent event processor.
Logically, it computes a function of the form

```
    f(event, old state) -> (actions, new state)
```

Obviously almost any computing model - including, but not limited
to, Urbit - can be defined in this form.  To create the illusion
of a computer that never loses state and never fails, we:

- log every event externally before it goes into u3
- keep a single reference to a permanent state noun.
- can abort any event without damaging the permanent state.
- snapshot the permanent state periodically, and/or prune logs.

### u3: the road model

`u3` uses a memory design which I'm sure someone has invented
somewhere before, because it's not very clever, but I've never
seen it anywhere in particular.

Every allocation starts with a solid block of memory, which `u3`
calls the `loom`.  How do we allocate on the loom?  You're
probably familiar with the Unix heap-stack design, in which the
stack grows downward and the heap (malloc arena) grows upward:

```
    0           brk                                          ffff
    |   heap     |                                    stack    |
    |------------#################################+++++++++++++|
    |                                             |            |
    0                                             sp         ffff
```

A road is a normal heap-stack system, except that the heap
and stack can point in **either direction**.  Therefore, inside
a road, we can nest another road in the **opposite direction**.

When the opposite road completes, its heap is left on top of
the opposite heap's stack.  It's no more than the normal
behavior of a stack machine for all subcomputations to push
their results on the stack.

The performance tradeoff of "leaping" - reversing directions in
the road - is that if the outer computation wants to preserve the
results of the inner one, not just use them for temporary
purposes, it has to **copy them**.

This is a trivial cost in some cases, a prohibitive cost in
others.  The upside, of course, is that all garbage accrued
in the inner computation is discarded at zero cost.

The goal of the road system is the ability to **layer** memory
models.  If you are allocating on a road, you have no idea
how deep within a nested road system you are - in other words,
you have no idea exactly how durable your result may be.
But free space is never fragmented within a road.

Roads do not reduce the generality or performance of a memory
system, since even the most complex GC system can be nested
within a road at no particular loss of performance - a road
is just a block of memory.

Each road (`u3a_road` to be exact) uses four pointers: `rut` is
the bottom of the arena, `hat` the top of the arena, `mat` the
bottom of the stack, `cap` the top of the stack.  (Bear in mind
that the road "stack" is not actually used as the C function-call
stack, though it probably should be.)

A "north" road has the stack high and the heap low:

```
    0           rut   hat                                    ffff
    |            |     |                                       |
    |~~~~~~~~~~~~-------##########################+++++++$~~~~~|
    |                                             |      |     |
    0                                            cap    mat  ffff
```

A "south" road is the other way around:

```
    0           mat   cap                                    ffff
    |            |     |                                       |
    |~~~~~~~~~~~~$++++++##########################------~~~~~|
    |                                             |      |     |
    0                                            hat    rut  ffff
```

Legend: `-` is durable storage (heap); `+` is temporary storage
(stack); `~` is deep storage (immutable); `$` is the allocation
frame `#` is free memory.

Pointer restrictions: pointers stored in `+` can point anywhere.
Pointers in `-` can only point to `-` or `~`; pointers in `~`
only point to `~`.

To "leap" is to create a new inner road in the `###` free space.
but in the reverse direction, so that when the inner road
"falls" (terminates), its durable storage is left on the
temporary storage of the outer road.

`u3` keeps a global variable, `u3_Road` or its alias `u3R`, which
points to the current road.  (If we ever run threads in inner
roads - see below - this will become a thread-local variable.)
Relative to `u3R`, `+` memory is called `junior` memory; `-`
memory is `normal` memory; `~` is `senior` memory.

### u3: explaining the road model

But... why?

We're now ready to understand why the road system works so
logically with the event and persistence model.

The key is that **we don't update refcounts in senior memory.**
A pointer from an inner road to an outer road is not counted.
Also, the outmost, or `surface` road, is the only part of the
image that gets checkpointed.

So the surface road contains the entire durable state of `u3`.
When we process an event, or perform any kind of complicated or
interesting calculation, **we process it in an inner road**.  If
its results are saved, they need to be copied.

Since processing in an inner road does not touch surface memory,
(a) we can leave the surface road in a read-only state and not
mark its pages dirty; (b) we can abort an inner calculation
without screwing up the surface; and (c) because inner results
are copied onto the surface, the surface doesn't get fragmented.

All of (a), (b) and (c) are needed for checkpointing to be easy.
It might be tractable otherwise, but easy is even better.

Moreover, while the surface is most definitely single-threaded,
we could easily run multiple threads in multiple inner roads
(as long as the threads don't have pointers into each others'
memory, which they obviously shouldn't).

Moreover, in future, we'll experiment more with adding road
control hints to the programmer's toolbox.  Reference counting is
expensive.  We hypothesize that in many - if not most - cases,
the programmer can identify procedural structures whose garbage
should be discarded in one step by copying the results.  Then,
within the procedure, we can switch the allocator into `sand`
mode, and stop tracking references at all.

### u3: rules for C programming

There are two levels at which we program in C: (1) above the
interpreter; (2) within the interpreter or jets.  These have
separate rules which need to be respected.

### u3: rules above the interpreter

In its relations with Unix, Urbit follows a strict rule of "call
me, I won't call you."  We do of course call Unix system calls,
but only for the purpose of actually computing.

Above Urbit, you are in a normal C/Unix programming environment
and can call anything in or out of Urbit.  Note that when using
`u3`, you're always on the surface road, which is not thread-safe
by default.  Generally speaking, `u3` is designed to support
event-oriented, single-threaded programming.

If you need threads which create nouns, you could use
`u3m_hate()` and `u3m_love()` to run these threads in subroads.
You'd need to make the global road pointer, `u3R`, a thread-local
variable instead.  This seems perfectly practical, but we haven't
done it because we haven't needed to.

### u3: rules within the interpreter

Within the interpreter, your code can run either in the surface
road or in a deep road.  You can test this by testing

```c
    (&u3H->rod_u == u3R)
```

ie: does the pier's home road equal the current road pointer?

Normally in this context you assume you're obeying the rules of
running on an inner road, ie, "deep memory."  Remember, however,
that the interpreter **can** run on surface memory - but anything
you can do deep, you can do on the surface.  The converse is by
no means the case.

In deep memory, think of yourself as if in a signal handler.
Your execution context is extremely fragile and may be terminated
without warning or cleanup at any time (for instance, by `ctrl-c`).

For instance, you can't call `malloc` (or C++ `new`) in your C
code, because you don't have the right to modify data structures
at the global level, and will leave them in an inconsistent state
if your inner road gets terminated.  (Instead, use our drop-in
replacements, `u3a_malloc()`, `u3a_free()`, `u3a_realloc()`.)

A good example is the different meaning of `c3_assert()` inside
and outside the interpreter.  At either layer, you can use
regular assert(), which will just kill your process.  On the
surface, `c3_assert()` will just... kill your process.

In deep execution, `c3_assert()` will issue an exception that
queues an error event, complete with trace stack, on the Arvo
event queue.   Let's see how this happens.

### u3: exceptions

You produce an exception with

```c
    /* u3m_bail(): bail out.  Does not return.
    **
    **  Bail motes:
    **
    **    %exit               ::  semantic failure
    **    %evil               ::  bad crypto
    **    %intr               ::  interrupt
    **    %fail               ::  execution failure
    **    %foul               ::  assert failure
    **    %need               ::  network block
    **    %meme               ::  out of memory
    **    %time               ::  timed out
    **    %oops               ::  assertion failure
    */
      c3_i
      u3m_bail(c3_m how_m);
```

Broadly speaking, there are two classes of exception: internal
and external.  An external exception begins in a Unix signal
handler.  An internal exception begins with a call to longjmp()
on the main thread.

There are also two kinds of exception: mild and severe.  An
external exception is always severe.  An internal exception is
normally mild, but some (like `c3__oops`, generated by
`c3_assert()`) are severe.

Either way, exceptions come with a stack trace.  The `u3` nock
interpreter is instrumented to retain stack trace hints and
produce them as a printable `(list tank)`.

Mild exceptions are caught by the first virtualization layer and
returned to the caller, following the behavior of the Nock
virtualizer `++mock` (in `hoon.hoon`)

Severe exceptions, or mild exceptions at the surface, terminate
the entire execution stack at any depth and send the cumulative
trace back to the `u3` caller.

For instance, `vere` uses this trace to construct a `%crud`
event, which conveys our trace back toward the Arvo context where
it crashed.  This lets any UI component anywhere, even on a
remote node, render the stacktrace as a consequence of the user's
action - even if its its direct cause was (for instance) a Unix
SIGINT or SIGALRM.

### u3: C structures on the loom

Normally, all data on the loom is nouns.  Sometimes we break this
rule just a little, though - eg, in the `u3h` hashtables.

To point to non-noun C structs on the loom, we use a `u3_post`,
which is just a loom word offset.  A macro lets us declare this
as if it was a pointer:

```c
    typedef c3_w       u3_post;
    #define u3p(type)  u3_post
```

Some may regard this as clever, others as pointless.  Anyway, use
`u3to()` and `u3of()` to convert to and from pointers.

When using C structs on the loom - generally a bad idea - make
sure anything which could be on the surface road is structurally
portable, eg, won't change size when the pointer size changes.
(Note also: we consider little-endian, rightly or wrongly, to
have won the endian wars.)
