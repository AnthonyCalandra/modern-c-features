# modern-c-features

## Overview

A collection of descriptions along with examples for C language and library features.

C17 contains defect reports and deprecations.

C11 includes the following language features:
- [generic selection](#generic-selection)
- [alignof](#alignof)
- [alignas](#alignas)
- [static_assert](#static_assert)
- [noreturn](#noreturn)
- [unicode literals](#unicode-literals)
- [anonymous structs and unions](#anonymous-structs-and-unions)

C11 includes the following library features:
- [bounds checking](#bounds-checking)
- [timespec_get](#timespec_get)
- [aligned_malloc](#aligned_malloc)
- [char32_t](#char32_t)
- [char16_t](#char16_t)
- [&lt;stdatomic.h&gt;](#stdatomic.h)
    - [Atomic types](#atomic-types)
    - [Atomic flags](#atomic-flags)
    - [Atomic variables](#atomic-variables)
- [&lt;threads.h&gt;](#threads.h)
    - [Threads](#threads)
    - [Mutexes](#mutexes)
    - [Condition variables](#condition-variables)
- [quick exiting](#quick-exiting)
- [exclusive mode file opening](#exclusive-mode-file-opening)

## C11 Language Features

### Generic selection
Using the `_Generic` keyword, select an expression based on the type of a given _controlling expression_. The format of `_Generic` is as follows:
```
_Generic( controlling-expression, T1: E1, ... )
```
where:
* `controlling-expresion` is an expression that yields a type present in the type list (`T1: E1, ...`).
* `T1` is a type.
* `E1` is an expression.

Optionally, specifying `default` in a type list, will match any controlling expression type.
```c
#define abs(expr) _Generic((expr), \
    int: abs(expr), \
    long int: labs(expr), \
    float: fabs(expr), \
    /* ... */ \
    /* Don't call abs for unsigned types, etc. */ \
    default: expr \
)

printf("%d %li %f %d\n", abs(-123), abs(-123l), abs(-3.14f), abs(123u));
// prints: 123 123 3.14 99 123
```

### alignof
Queries the _alignment requirement_ of a given type using the `_Alignof` keyword or `alignof` convenience macro defined in `<stdalign.h>`.
```c
// On a 64-bit x86 machine:
alignof(char); // == 1
alignof(int); // == 4
alignof(int*); // == 8

// Queries alignment of array members.
alignof(int[5]); // == 4

struct foo { char a; char b; };
alignof(struct foo); // == 1

// 3 bytes of padding between `a` and `b`.
struct bar { char a; int b; };
alignof(struct bar); // == 4
```

`max_align_t` is a type whos alignment is as large as that of every scalar type.

### alignas
Sets the alignment of the given object using the `_Alignas` keyword, or `alignof` macro in `<stdalign.h>`.
```c
struct sse_t
{
    // Aligns `sse_data` on a 16 byte boundary.
    alignas(16) float sse_data[4];
};

struct buffer
{
    // Align `buffer` to the same alignment boundary as an `int`.
    alignas(int) char buf[sizeof(int)];
};
```

`max_align_t` is a type whos alignment is as large as that of every scalar type.

### static_assert
Compile-time assertion either using the `_Static_assert` keyword or the `static_assert` keyword macro defined in `assert.h`.
```c
static_assert(sizeof(int) == sizeof(char), "`int` and `char` sizes do not match!");
```

### noreturn
Specifies the function does not return. If the function returns, either by returning via `return` or reaching the end of the function body, its behaviour is undefined. `noreturn` is a keyword macro defined in `<stdnoreturn.h>`.
```c
noreturn void foo()
{
    exit(0);
}
```

### Unicode literals
Create 16-bit or 32-bit Unicode string literals and character constants.
```c
char16_t c1 = u'貓';
char32_t c2 = U'🍌';

char16_t s1[] = u"a猫🍌"; // => [0x0061, 0x732B, 0xD83C, 0xDF4C, 0x0000]
char32_t s2[] = U"a猫🍌"; // => [0x00000061, 0x0000732B, 0x0001F34C, 0x00000000]
```

See: [char32_t](#char32_t), [char16_t](#char16_t).

### Anonymous structs and unions
Allows unnamed (i.e. _anonymous_) structs or unions. Every member of an anonymous union is considered to be a member of the enclosing struct or union, keeping the layout intact. This applies recursively if the enclosing struct or union is also anonymous.
```c
struct v
{
   union // anonymous union
   {
       int a;
       long b;
   };
   int c;
} v;
 
v.a = 1;
v.b = 2;
v.c = 3;

printf("%d %ld %d", v.a, v.b, v.c); // prints "2 2 3"
```
```c
union v
{
   struct // anonymous struct
   {
       int a;
       long b;
   };
   int c;
} v;
 
v.a = 1;
v.b = 2;
v.c = 3;

printf("%d %ld %d", v.a, v.b, v.c); // prints "3 2 3"
```

## C11 Library Features

### Bounds checking
Many standard library functions now have bounds-checking versions of existing functions. You can specify a callback function using `set_constraint_handler_s` as a _constraint handler_ when a function fails its boundary check. The standard library includes two constraint handlers: `abort_handler_s` which writes to `stderr` and terminates the program; and `ignore_handler_s` ignores the violation and continue the program.

Standard library functions with bounds-checked equivalents will be appended with `_s`. For example, some bounds-checked versions include `fopen_s`, `get_s`, `asctime_s`, etc.

Example of a custom constraint handler with `get_s`:
```c
void custom_handler_s(const char* restrict msg, void* restrict ptr, errno_t error)
{
    fprintf(stderr, "ERROR: %s\n", msg);
    abort();
}

set_constraint_handler_s(custom_handler_s);
char buffer[BUFFER_SIZE];
gets_s(buffer, BUFFER_SIZE);
printf("Entered: %s\n", buffer);
```
If a user enters a string greater than or equal to `BUFFER_SIZE` the custom constraint handler will be called.

### timespec_get
Populates a `struct timespec` object with the current time given a time base.
```c
struct timespec ts;
timespec_get(&ts, TIME_UTC);
char buff[BUFFER_SIZE];
strftime(buff, sizeof(buff), "%D %T", gmtime(&ts.tv_sec));
printf("Current time: %s.%09ld UTC\n", buff, ts.tv_nsec);
```

### aligned_malloc
Allocates bytes of storage whose alignment is specified.
```c
// Allocate at a 256-byte alignment.
int* p = aligned_alloc(256, sizeof(int));
// ...
free(p);
```

### char32_t
An unsigned integer type for holding 32-bit wide characters.

See: [Unicode literals](#unicode-literals).

### char16_t
An unsigned integer type for holding 16-bit wide characters.

See: [Unicode literals](#unicode-literals).

### &lt;stdatomic.h&gt;

#### Atomic types
The `_Atomic` type specifier/qualifier is used to denote that a variable is an _atomic type_. An _atomic type_ is treated differently than non-atomic types because access to atomic types provide freedom from _data races_. For example, the following code was compiled on x86 clang 4.0.0:
```c
_Atomic int a;
a = 1; // mov     ecx, 1
       // xchg    dword ptr [rbp - 8], ecx

int b;
b = 1; // mov     dword ptr [rbp - 8], 1
```
Notice that the atomic type used a `mov` then an `xchg` instruction to assign the value `1` to `a`, while `b` uses a single `mov`. Other operations may include `+=`, `++`, and so on... Ordinary read and write access to atomic types are _sequentially-consistent_.

See [this StackOverflow post](https://stackoverflow.com/a/26463282/3229983) for more information.

A variety of typedefs exist which expand to `_Atomic T`. For example, `atomic_int` is typedef'd to `_Atomic int`. See [this list](http://en.cppreference.com/w/c/atomic) for a more complete list.

Atomic types can also be tested for lock-freedom by using the `atomic_is_lock_free` function or the various `ATOMIC_*_LOCK_FREE` macro constants. Since `atomic_flag`s are guaranteed to be lock-free, the following example will always return `1`:
```c
atomic_flag flag = ATOMIC_FLAG_INIT;
atomic_is_lock_free(&flag); // == 1
```

#### Atomic flags
An `atomic_flag` is a lock-free (guaranteed), atomic boolean type representing a flag. An example of an atomic operation which uses a flag is test-and-set. Initialize `atomic_flag` objects using the `ATOMIC_FLAG_INIT` macro.

A simple spinlock implementation using `atomic_flag` as the "spin value":
```c
struct spinlock
{
    // false - lock is free, true - lock is taken
    atomic_flag flag;
};

void acquire_spinlock(struct spinlock* lock)
{
    // `atomic_flag_test_and_set` returns the value of the flag.
    // We keep spinning until the lock is free (value of the flag is `false`).
    while (atomic_flag_test_and_set(&lock->flag) == true);
}

void release_spinlock(struct spinlock* lock)
{
    atomic_flag_clear(&lock->flag);
}

void print_foo(void* lk)
{
    struct spinlock* lock = (struct spinlock*) lk;
    acquire_spinlock(lock);
    printf("foo\n");
    release_spinlock(lock);
}

void print_bar(void* lk)
{
    struct spinlock* lock = (struct spinlock*) lk;
    acquire_spinlock(lock);
    printf("bar\n");
    release_spinlock(lock);
}
```
```c
struct spinlock lock = { .flag = ATOMIC_FLAG_INIT };

// In Thread A:
print_foo(&lock);
// ==============
// In Thread B:
print_bar(&lock);
```

#### Atomic variables
An _atomic variable_ is a variable which is declared as an _atomic type_. Atomic variables are meant to be used with the atomic operations which operate on the values held by these variables (except for the flag operations, ie. `atomic_flag_test_and_set`). Unlike `atomic_flag`, these are not guaranteed to be lock-free. Initialize atomic variables using the `ATOMIC_VAR_INIT` macro or `atomic_init` if it has been default-constructed already.

The C11 Atomics library provides many additional atomic operations, memory fences, and allows specifying memory orderings for atomic operations.
```c
struct spinlock
{
    // false - lock is free, true - lock is taken
    atomic_bool flag;
};

void acquire_spinlock(struct spinlock* lock)
{
    bool expected = false;
    // `atomic_compare_exchange_weak` returns `false` when the value of the
    // flag is not equal to `desired`. We keep spinning until the lock is
    // free (value of the flag is `false`).
    while (atomic_compare_exchange_weak(&lock->flag, &expected, true) == false)
    {
        // `expected` will get set to the value of the flag for every call.
        // Reset it since we always "expect" `false`.
        expected = false;
    }
}

void release_spinlock(struct spinlock* lock)
{
    atomic_store(&lock->flag, false);
}

void print_foo(void* lk)
{
    struct spinlock* lock = (struct spinlock*) lk;
    acquire_spinlock(lock);
    printf("foo\n");
    release_spinlock(lock);
}

void print_bar(void* lk)
{
    struct spinlock* lock = (struct spinlock*) lk;
    acquire_spinlock(lock);
    printf("bar\n");
    release_spinlock(lock);
}
```
```c
struct spinlock lock;
atomic_init(&lock.flag, false);

// In Thread A:
print_foo(&lock);
// ==============
// In Thread B:
print_bar(&lock);
```

### &lt;threads.h&gt;
C11 provides an OS-agnostic thread library supporting thread creation, mutexes, and condition variables. Threading library is located in `<threads.h>`. As of September 2023, it is poorly supported in most major compilers.

#### Threads
Creates a thread and executes `print_n` in the new thread.
```c
void print_n(int* n)
{
    printf("%d\n", *n); // prints "123"
}

int n = 123;
thrd_t thr;
const int ret = thrd_create(
    &thr, (thrd_start_t) &print_n, (void*) &n);
thrd_join(thr, NULL);
```

#### Mutexes
C11 provides mutexes that support:
* Timed locking - Given a `timespec` object, blocks the current thread until the mutex is locked or until it times out.
* Recursive locking - Supports recursive locking (e.g. re-locking in recursive functions).
Recursive timed locks can also be created, capturing both these operations.

Specify which type of mutex to be created by passing any of `mtx_plain`, `mtx_timed`, `mtx_recursive`, or any combination of these to `mtx_init`'s `type` parameter.

```c
mtx_t mutex;
const int ret = mtx_init(&mutex, mtx_plain);

// In each thread: =============
mtx_lock(&mutex);
// Critical section
mtx_unlock(&mutex);
// ==============================

mtx_destroy(&mutex);
```

Mutexes must be cleaned up by calling `mtx_destroy`.

#### Condition variables
C11 provides condition variables as part of its concurrency library. Condition variables will block when waiting, support timed-waiting, and can be signalled and broadcasted. Note that spurious wakeups may occur.

Condition variables must be cleaned up by calling `cnd_destroy`.
```c
// Assume remove_from_queue, add_to_queue, and
// can_consume are defined.

mtx_t mutex;
// Initialize mutex...

cnd_t cond;
const int ret = cnd_init(&cond);

// In a consumer thread: =====
// Check in a loop due to spurious wakeups.
while (!can_consume && cnd_wait(&cond, &mutex));
remove_from_queue();
// =========================

// In a producer thread: =====
add_to_queue();
can_consume = true;
cnd_signal(&cond);
// ===========================

cnd_destroy(&cond);
```

### Quick exiting
Causes program termination to occur where clean termination is not possible or impractical; for example if cooperative cancellation between different threads is not possible or cancellation order is unachievable. This would have resulted in some threads trying to access static-duration objects while they are/have been destructed. 

Quick exiting allows applications to register handlers to be called on exit while being able to access static duration objects (unlike `atexit`). Quick exiting does not execute static-duration object destructors in order to achieve this.

Therefore, quick exiting can be thought of as being "in between" choosing to normally terminate a program, and abnormally terminating using `abort`.

```c
void f()
{
    // called first
}
 
void g()
{
    // called second
}
 
int main()
{
    at_quick_exit(f);
    at_quick_exit(g);
    quick_exit(0);
}
```

### Exclusive mode file opening
File access mode flag `x` can optionally be appended to `w` or `w+` specifiers. This flag forces the function to fail if the file exists, instead of overwriting it.
```c
FILE* fp = fopen(fname, "w+x");
if (!fp)
{
    // File either exists or there was an error
}
else
{
    // File opened successfully.
    fclose(fp);
}
```

## Author

Anthony Calandra

## License

MIT
