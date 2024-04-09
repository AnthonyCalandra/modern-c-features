# modern-c-features

## Overview

A collection of descriptions along with examples for C language and library features.

C23 includes the following language features:
- [auto](#auto)
- [constexpr](#constexpr)
- [decimal floating-point types](#decimal-floating-point-types)
- [bit-width integers](#bit-width-integers)
- [binary literals](#binary-literals)
- [char8_t](#char8_t)
- [utf-8 character literals](#utf-8-character-literals)
- [unicode string literals](#unicode-string-literals)
- [empty initializer](#empty-initializer)
- [attributes](#attributes)
- [new keywords (C23)](#new-keywords-c23)
- [nullptr](#nullptr)
- [`#embed`](#embed)
- [enums with underlying type](#enums-with-underlying-type)
- [typeof](#typeof)
- [improved compatibility for tagged types](#improved-compatibility-for-tagged-types)

C23 includes the following library features:
- [floating-point formatting functions](#floating-point-formatting-functions)
- [memset_explicit](#memset_explicit)
- [`unreachable` macro](#unreachable-macro)
- [`memccpy`](#memccpy)
- [`strdup` and `strndup`](#strdup-and-strndup)
- [`gmtime_r` and `localtime_r`](#gmtime_r-and-localtime_r)
- [`timespec_getres`](#timespec_getres)

C17 contains defect reports and deprecations.

C11 includes the following language features:
- [generic selection](#generic-selection)
- [alignof](#alignof)
- [alignas](#alignas)
- [static_assert](#static_assert)
- [noreturn](#noreturn)
- [wide string literals](#wide-string-literals)
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

## C23 Language Features

### auto
`auto`-typed variables are deduced by the compiler according to the type of their initializer.
```c
auto f = 123.0f; // `f` deduced to `float`

#include <tgmath.h> // import the `cos` macro
auto c = cos(x); // `c` deduced to a type depending on `x`

#define div(X, Y)            \
  _Generic((X)+(Y),          \
           int: div,         \
           long: ldiv,       \
           long long: lldiv) \
           ((X), (Y))
auto z = div(x, y); // deduced to either `int`, `long`, `long long`
```

See: [generic selection](#generic-selection).

### constexpr
Scalars specified with `constexpr` are constants: they cannot be modified after being initialized, and must be fully initialized to a value that can be stored in the given type.

The C `constexpr` specifier does not support functions, structures, unions, or arrays. Additionally, `constexpr` cannot be specified for pointer, atomic, or volatile-qualified types.

```c
constexpr size_t cache_line_size_bytes = 64;
```

### Decimal floating-point types
Supports IEEE-754 Decimal floating-point types, `_Decimal32`, `_Decimal64`, `_Decimal128`. These floating point types are designed for base-10 floating point semantics.

As of September 2023, compiler support is lacking for most features.
```c
_Decimal32 decsum = 0.0df;
for (int i = 0; i < 10; i++)
    decsum += 0.1df;
printf("%.17Ha\n", decsum); // would print "1.0"

float fsum = 0.0f;
for (int i = 0; i < 10; i++)
    fsum += 0.1f;
printf("%.17f\n", fsum); // could print "1.00000011920928955"
```

### Bit-width integers
The `_BitInt(N)` type allows specifying an `N` bit integer signed or unsigned type.
```c
_BitInt(4) sbi; // 4-bit width signed integer
unsigned _BitInt(4) ubi; // 4-bit width unsigned integer
```

### Binary literals
Binary literals provide a convenient way to represent a base-2 number. It is possible to separate digits with `'`.
```c
0b110 // == 6
0b1111'1111 // == 255
```

### char8_t
An unsigned char type for holding 8-bit wide characters.

See: [char32_t](#char32_t), [char16_t](#char16_t), [unicode string literals](#unicode-string-literals).

### UTF-8 character literals
A character literal that begins with `u8` is a character literal of type `char8_t`. The value of a UTF-8 character literal is equal to its ISO 10646 code point value.
```c
char8_t x = u8'x';
```

See: [char8_t](#char8_t), [unicode string literals](#unicode-string-literals).

### Unicode string literals
String literals prefixed with `u8`, `u`, and `U` represent UTF-8, UTF-16, and UTF-32 strings respectively.

See: [char32_t](#char32_t), [char16_t](#char16_t), [char8_t](#char8_t), [wide string literals](#wide-string-literals).

### Empty initializer
An object is empty-initialized if it is explicitly initialized using only a pair of braces. Arrays of unknown size cannot be initialized by an empty initializer. If an object is initialized using an empty initializer, default initialization occurs:
* Pointers are initialized to `NULL`;
* Decimal floating point types are initialized to positive zero;
* Arithmetic types are initialized to zero;
* Array types are initialized such that each slot is initialized according to these rules;
* Aggregate types are initialized such that each member is initialized according to these rules;
* Unions are initialized such that the first named member is initialized according to these rules.

```c
char c = {}; // == 0
float f = {}; // == 0

struct
{
    int x;
    int y;
} f = {}; // == { x: 0, y: 0}

union
{
        char x;
        double y;
} u = {}; // u.x == 0

int ia[5] = {}; // == [0, 0, 0, 0, 0]
```

### Attributes
Attributes provide a universal syntax over `__attribute__(...)`, `__declspec`, etc. C++-styled attributes.

C23 provides the following attributes:
* `[[deprecated("reason")]]` - the entity declared with the attribute (such as a function), is deprecated. Specifying a reason is optional and can be ommitted.
* `[[fallthrough]]` - indicates falling-through in a switch statement case is intentional.
* `[[nodiscard("reason")]]` - encourages a compiler warning if the return value for the function declared with this attribute is discarded. Specifying a reason is optional and can be omitted.
* `[[maybe_unused]]` - suppresses a compiler warning on the entity where this attribute is declared if it's unused (such as an unused function parameter), if applicable.
* `[[noreturn]]` - indicates that the function this attribute is declared on does not return.
* Compiler-specific directives - such as `[[clang::no_sanitize]]`.

```c
// `noreturn` attribute indicates `f` doesn't return.
[[noreturn]] void f()
{
    exit(0);
}
```

See: [noreturn](#noreturn).

### New keywords (C23)
New keywords replacing traditional macros definitions:
* `true` and `false`
* `thread_local`
* `static_assert`

See: [static_assert](#static_assert).

### nullptr
Introduces a new null pointer type designed to replace `NULL`. `nullptr` itself is of type `nullptr_t` which can be implicitly converted into pointer types and `bool`, and -- unlike `NULL` -- is not convertible to integral types.
```c
void foo(int);
foo(NULL); // valid
foo(nullptr); // error
```

### `#embed`
`#embed` is a preprocessor directive to include text and binary resources directly into source code. This pattern is common in applications such as games that create custom fonts, load images into memory, etc. This allows the C preprocessor to replace external tools that convert resources to byte representations as C arrays.

The `#embed` preprocessor directive also includes optional parameters including `suffix`, `prefix`, and `if_empty`.
```c
const uint8_t image_bytes[] = {
#embed "image.bmp"
};
 
const char message_text[] = {
#embed "message.txt" if_empty('M', 'i', 's', 's', 'i', 'n', 'g', '\n')
,'\0'
};
```

### Enums with underlying type
C23 enums provide the ability to optionally specify the underlying type.
```c
enum e : unsigned short
{
    x // `x` is an `unsigned short`
};
```

### typeof
Gets the type of an expression, similar to `decltype` in C++. Also includes `typeof_unqual` which removes cv-qualifiers from the type.
```c
int a;
const volatile int b;
typeof(a) c; // has type of `int`
typeof_unqual(b) d; // has type of `int
```

### Improved compatibility for tagged types
Tagged types that have the same tag name and content become compatible not only across translation units but also inside the same translation unit. Also, redeclaration of the same tagged types is now allowed.
```c
// header
#define PRODUCT(A ,B) struct prod { A a; B b; }
#define SUM(A, B) struct sum { bool flag; union { A a; B b; }; }

// source
void foo(PRODUCT(int, SUM(float, double)) x)
{
    // ...
}

void bar(PRODUCT(int, SUM(float, double)) y)
{
    foo(y); // compatible type -- compiles
}
```

## C23 Library Features

### Floating-point formatting functions
Converts floating-point values to byte strings. Convert `floats`, `doubles`, and `long double`s using `strfromf`, `strfromd`, and `strfroml` respectively.
```c
char buf[BUFFER_SIZE] = {};
strfromf(&buf, BUFFER_SIZE, "%f", 123.0f);
```

### memset_explicit
Copies the given value into each of the first `count` characters of the object pointed to by `dest`. Unlike `memset`, this function cannot be optimized away by the compiler; therefore, this function is guaranteed to perform the memory write.
```c
char str[] = "foo";
memset_explicit(str, 0, sizeof(str));
```

### `unreachable` macro
Provides a standard macro for (historically) compiler-specific macros for denoting an unreachable area of the code.
```c
if (1 > 0) ...
else unreachable();
```

### `memccpy`
Copies bytes from source to destination stopping after either the terminating byte is found (and also copied) or the given number of bytes is copied.
```c
// Assume `src` is populated elsewhere.
char dest[MAX_LEN] = {};
// Copy to `dest` until either the NUL terminator (zero) byte
// is found or we are at `MAX_LEN - 1` bytes copied.
memccpy(dest, src, 0, MAX_LEN - 1);
```

### `strdup` and `strndup`
These functions duplicate a given string by allocating a buffer and copying. The returned string is always null-terminated. Be sure to free the memory returned by thhese functions after use.
```c
const char* src = "foobarbaz";

char* src2 = strdup(src); // "foobarbaz"
free(src2);

char* src3 = strndup(src, 3); // "foo"
free(src3);
```

### `gmtime_r` and `localtime_r`
Works as `gmtime` and `localtime` does but uses a given storage buffer for the result. Returns a `NULL` pointer on error.
```c
time_t t = time(NULL);

struct tm buf;
struct* tm ret = gmtime_r(&t, &buf);
// ...
struct* tm ret2 = localtime_r(&t, &buf);
```

### `timespec_getres`
Stores the resolution of time provided by the given base. Returns zero on failure.
```c
struct timespec ts;
const int res = timespec_getres(&ts, TIME_UTC);
if (res == TIME_UTC)
{
    // ...
}
```

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

See: [new keywords (C23)](#new-keywords-c23).

### noreturn
Specifies the function does not return. If the function returns, either by returning via `return` or reaching the end of the function body, its behaviour is undefined. `noreturn` is a keyword macro defined in `<stdnoreturn.h>`.
```c
noreturn void foo()
{
    exit(0);
}
```

See: [attributes](#attributes).

### Wide string literals
Create 16-bit or 32-bit wide string literals and character constants.
```c
char16_t c1 = u'貓';
char32_t c2 = U'🍌';

char16_t s1[] = u"a猫🍌"; // => [0x0061, 0x732B, 0xD83C, 0xDF4C, 0x0000]
char32_t s2[] = U"a猫🍌"; // => [0x00000061, 0x0000732B, 0x0001F34C, 0x00000000]
```

See: [char32_t](#char32_t), [char16_t](#char16_t), [unicode string literals](#unicode-string-literals).

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

See: [wide string literals](#wide-string-literals), [char16_t](#char16_t), [unicode string literals](#unicode-string-literals).

### char16_t
An unsigned integer type for holding 16-bit wide characters.

See: [wide string literals](#wide-string-literals), [char32_t](#char32_t), [unicode string literals](#unicode-string-literals).

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
