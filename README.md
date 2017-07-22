# modern-c-features

## Overview

A collection of descriptions along with examples for C language and library features.

C11 includes the following language features:
- [generic selection](#generic-selection)
- [alignof](#alignof)
- [static_assert](#static_assert)
- [noreturn](#noreturn)

C11 includes the following library features:
- [bounds checking](#bounds-checking)
- [timespec_get](#timespec_get)
- [char32_t](#char32_t)
- [char16_t](#char16_t)
- [&lt;stdatomic.h&gt;](#stdatomic.h)
    - [atomic types](#atomic-types)
    - [atomic flags](#atomic-flags)
    - [atomic vars](#atomic-vars)
- [&lt;threads.h&gt;](#threads.h)
    - [threads](#threads)
    - [mutexes](#mutexes)
    - [condition variables](#condition-variables)

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
    /* Don't call abs for unsigned types, etc. */ default: expr \
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

### static_assert
Compile-time assertion either using the `_Static_assert` keyword or the `static_assert` keyword macro defined in `assert.h`.
```c
#define static_assert(e, m) _Static_assert(e, m)
static_assert(sizeof(int) == sizeof(char), "`int` and `char` sizes do not match!");
```

### noreturn
Specifies the function does not return. If the function returns, either by returning via `return` or reaching the end of the function body, behaviour is undefined. `noreturn` is a keyword macro defined in `<stdnoreturn.h>`.
```c
#define noreturn _Noreturn
noreturn void foo() {
    exit(0);
}
```

## C11 Library Features

### Bounds checking
Many standard library functions now have bounds-checking versions of existing functions. You can specify a callback function using `set_constraint_handler_s` as a _constraint handler_ when a function fails its boundary check. The standard library includes two constraint handlers: `abort_handler_s` which writes to `stderr` and terminates the program; and `ignore_handler_s` ignores the violation and continue the program.

Standard library functions with bounds-checked equivalents will be appended with `_s`. For example, some bounds-checked versions include `fopen_s`, `get_s`, `asctime_s`, etc.

Example of a custom constraint handler with `get_s`:
```c
#define BUFFER_SIZE 3

void custom_handler_s(const char *restrict msg, void *restrict ptr, errno_t error) {
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
char buff[100];
strftime(buff, sizeof(buff), "%D %T", gmtime(&ts.tv_sec));
printf("Current time: %s.%09ld UTC\n", buff, ts.tv_nsec);
```

### char32_t
An unsigned integer type for holding 32-bit wide characters.

### char16_t
An unsigned integer type for holding 16-bit wide characters.

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
struct spinlock {
    // false - lock is free, true - lock is taken
    volatile atomic_flag flag;
};

#define SPINLOCK_INIT { .flag = ATOMIC_FLAG_INIT }

void acquire_spinlock(struct spinlock* lock) {
    // `atomic_flag_test_and_set` returns the value of the flag.
    // We keep spinning until the lock is free (value of the flag is `false`).
    while (atomic_flag_test_and_set(&lock->flag) == true);
}

void release_spinlock(struct spinlock* lock) {
    atomic_flag_clear(&lock->flag);
}

void* print_foo(void* l) {
    struct spinlock* lock = (struct spinlock*) l;
    acquire_spinlock(lock);
    sleep(3);
    printf("foo\n");
    release_spinlock(lock);
    return NULL;
}

void* print_bar(void* l) {
    struct spinlock* lock = (struct spinlock*) l;
    acquire_spinlock(lock);
    printf("bar\n");
    release_spinlock(lock);
    return NULL;
}

pthread_t thread1, thread2;
struct spinlock lock = SPINLOCK_INIT;
pthread_create(&thread1, NULL, print_foo, (void*) &lock);
pthread_create(&thread2, NULL, print_bar, (void*) &lock);
pthread_join(thread1, NULL);
pthread_join(thread2, NULL);
```
Run the example until `thread1` is scheduled first. A three second pause will occur before `foo` is printed, then subsequently `bar` will print.

#### Atomic vars
An _atomic variable_ is a variable which is declared as an _atomic type_. Atomic variables are meant to be used with the atomic operations which operate on the values held by these variables (except for the flag operations, ie. `atomic_flag_test_and_set`). Unlike `atomic_flag`, these are not guaranteed to be lock-free. Initialize atomic variables using the `ATOMIC_VAR_INIT` macro or `atomic_init` if it has been default-constructed already.

```c
struct spinlock {
    // false - lock is free, true - lock is taken
    volatile atomic_bool flag;
};

#define SPINLOCK_INIT { .flag = ATOMIC_VAR_INIT(false) }

void acquire_spinlock(struct spinlock* lock) {
    bool expected = false;
    // `atomic_compare_exchange_weak` returns `false` when the value of the flag is not equal to `desired`.
    // We keep spinning until the lock is free (value of the flag is `false`).
    while (atomic_compare_exchange_weak(&lock->flag, &expected, true) == false) {
        // `expected` will get set to the value of the flag for every call. Reset it since we always "expect" `false`.
        expected = false;
    }
}

void release_spinlock(struct spinlock* lock) {
    atomic_store(&lock->flag, false);
}

void* print_foo(void* l) {
    struct spinlock* lock = (struct spinlock*) l;
    acquire_spinlock(lock);
    sleep(3);
    printf("foo\n");
    release_spinlock(lock);
    return NULL;
}

void* print_bar(void* l) {
    struct spinlock* lock = (struct spinlock*) l;
    acquire_spinlock(lock);
    printf("bar\n");
    release_spinlock(lock);
    return NULL;
}

pthread_t thread1, thread2;
struct spinlock lock = SPINLOCK_INIT;
pthread_create(&thread1, NULL, print_foo, (void*) &lock);
pthread_create(&thread2, NULL, print_bar, (void*) &lock);
pthread_join(thread1, NULL);
pthread_join(thread2, NULL);
```

### &lt;threads.h&gt;

#### Threads
TODO

#### Mutexes
TODO

#### Condition variables
TODO

## Author

Anthony Calandra

## License

MIT
