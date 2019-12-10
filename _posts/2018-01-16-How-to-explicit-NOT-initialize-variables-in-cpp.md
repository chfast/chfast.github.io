---
layout: single
title: How to explicit NOT initialize variables in C++
---

I noticed a pattern in code fragments interacting with functions having _output_
parameters. Usually, the such functions can be found in C APIs where output
parameters are used to "return" multiple values or a value of a struct type.

In case the function has pure output parameters (the value is never read by
the function) you may want leave it uninitialized because any value is going
to be overwritten anyway.

The following code snippet shows a small example.

```cpp
void read_number(int* x);

int example()
{
    int x;
    read_number(&x);
    return x;
}
```

Compilers are usually fine with this unless they can prove that the value can
be read uninitialized. But some static analysis tools, linters or humans may
get suspicious. I want to find a way to explicit mark the fact the variable `x`
may stay uninitialized before calling `read_number()`.

The D language has a syntax feature called [Void Initialization] to address
this problem. The above example would look like this in D:

```d
void read_number(int* x);

int example()
{
    int x = void;
    read_number(&x);
    return x;
}
```


## Introducing uninitialized_t

In C++ we can create a struct type implicit convertible to (almost) any type.

```cpp
struct uninitialized_t
{
    template<typename T>
    operator T() const noexcept
    {
        char data alignas(T)[sizeof(T)];
        return *reinterpret_cast<T*>(data);
    }
};

static constexpr uninitialized_t uninitialized;
```

The modified example using `uninitialized_t` would look like this:

```cpp
void read_number(int* x);

void example_uninitialized_int()
{
    int x = uninitialized_t{};
    read_number(&x);
}
```

With the help of `constexpr uninitialized_t uninitialized` we can make it look
more like the `uninitialized` being a keyword.

```cpp
void read_number(int* x);

void example_uninitialized_int()
{
    int x = uninitialized;
    read_number(&x);
}
```

The `uninitialized` works also for user-defined types with non-default
constructors. The type must be _movable_.

```cpp
struct Complex
{
    double re = 0.0 / 0.0;  ///< Init with NaN.
    double im = 0.0 / 0.0;  ///< Init with NaN.
};


void read_complex(Complex* z);


Complex example_uninitialized_complex()
{
    Complex z = uninitialized;
    read_complex(&z);
    return z;
}
```


## The implementation

The type `uninitialized_t` makes itself convertible by implementing the
template converting operator. The operator is implemented like this:

1. Fist we create an uninitialized array of chars matching the size of the
   destination type.
2. The array is also create with the alignment matching the alignment of
   the destination type using C++11 feature `alignas(T)`.
3. Lastly, the data is casted to the destination type with `reinterpret_cast`.


## Does is actually work?

Let's add 2 more examples where we properly initialize `int x` and `Complex z`.

```cpp
int example_initialized_int()
{
    int x = 0;
    read_number(&x);
    return x;
}

Complex example_initialized_complex()
{
    Complex z;
    read_complex(&z);
    return z;
}
```

Using clang 5.0 with arguments `-O3 -std=c++17 -march=skylake` we get the
following x86_64 assembly:

```asm
example_initialized_int(): # @example_initialized_int()
  push rax
  mov dword ptr [rsp + 4], 0
  lea rdi, [rsp + 4]
  call read_number(int*)
  mov eax, dword ptr [rsp + 4]
  pop rcx
  ret

example_uninitialized_int(): # @example_uninitialized_int()
  push rax
  lea rdi, [rsp + 4]
  call read_number(int*)
  mov eax, dword ptr [rsp + 4]
  pop rcx
  ret

.LCPI2_0:
  .quad 9221120237041090560 # double NaN
  .quad 9221120237041090560 # double NaN
example_initialized_complex(): # @example_initialized_complex()
  sub rsp, 24
  vmovaps xmm0, xmmword ptr [rip + .LCPI2_0] # xmm0 = [nan,nan]
  vmovaps xmmword ptr [rsp], xmm0
  mov rdi, rsp
  call read_complex(Complex*)
  vmovsd xmm0, qword ptr [rsp] # xmm0 = mem[0],zero
  vmovsd xmm1, qword ptr [rsp + 8] # xmm1 = mem[0],zero
  add rsp, 24
  ret

example_uninitialized_complex(): # @example_uninitialized_complex()
  sub rsp, 24
  lea rdi, [rsp + 8]
  call read_complex(Complex*)
  vmovsd xmm0, qword ptr [rsp + 8] # xmm0 = mem[0],zero
  vmovsd xmm1, qword ptr [rsp + 16] # xmm1 = mem[0],zero
  add rsp, 24
  ret
```

In case of `int x` the usage of ` = uninitialized` is saving a single
instruction `mov dword ptr [rsp + 4], 0` that is setting 4 bytes on the stack
to zero.

For `Complex` initialization the compiler copies two NaN values from data
section in place of the `Complex` instance. This is done with two `vmovaps`
AVX instructions.

```asm
vmovaps xmm0, xmmword ptr [rip + .LCPI2_0] # xmm0 = [nan,nan]
vmovaps xmmword ptr [rsp], xmm0
```

These two can be eliminated with `= uninitialized`.

[Void Initialization]: https://dlang.org/spec/declaration.html#void_init
