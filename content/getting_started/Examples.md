---
title: "C++ Examples"
date: 2023-07-25
draft: false
weight: 25
katex: true
---

There are many ways to implement a given calculation. This document contains examples of how to apply Enzyme to different kinds of patterns that show up frequently in C++. To make the code snippets below more concise, we'll assume that each example snippet has the following definitions prepended to the translation unit:

```cpp
int enzyme_dup;
int enzyme_dupnoneed;
int enzyme_out;
int enzyme_const;

template < typename return_type, typename ... T >
return_type __enzyme_fwddiff(void*, T ... );

template < typename return_type, typename ... T >
return_type __enzyme_autodiff(void*, T ... );
```



## Free Functions: Return-By-Value

Consider a function with one scalar input and one scalar output, \\(f(x) \coloneqq x^2.\\) We can implement this in C++ as a free function:

```cpp
double f(double x) { return x * x; }
```

The derivative of this function is simply \\(f'(x) = 2x\\), but rather than implement that by hand, let's see how do it with automatically with Enzyme:

```cpp
double x = 5.0;
double dx = 1.0;
double df_dx = __enzyme_fwddiff<double>((void*)f, enzyme_dup, x, dx); 
printf("f(x) = %f, f'(x) = %f", f(x), df_dx);
// prints f(x) = 25.000000, f'(x) = 10.000000
```

The first argument tells enzyme which function we want to differentiate, and the subsequent 
arguments describe where to evaluate the derivatve and in what "direction".

Link to example: https://fwd.gymni.ch/Hx6hwt

--------

Next, we'll look at a function with two inputs and one output,
$$
f(x, y) \coloneqq x \\, y + \frac{1}{y}
$$

The "derivative" (i.e. the Jacobian) of this function is a row vector with two entries:
$$
\bold{J} = \left[
\begin{array}{cc}
\displaystyle \frac{\partial f}{\partial x} & \displaystyle \frac{\partial f}{\partial y}
\end{array}
\right]
$$
There are two fundamental operations that involve the Jacobian:

1. "forward mode" differentiation: compute \\(df := \bold{J} \cdot d\bold{x}\\), (where \\(d\bold{x} = [dx \\,\\,  dy]^\top\\))
2. "reverse mode" differentiation: compute \\(\boldsymbol{\mu} := \bold{J}^\top  \lambda\\\)

There's lots more to say here about when to use forward mode or reverse mode, but that's beyond the scope of this document, so we'll just focus on how to use Enzyme to perform these two kinds of differentiation for us.

Like before, start by implementing the original function

```cpp
double f(double x, double y) { return x * y + 1.0 / y; }
```

#### Forward Mode

`__enzyme_fwddiff` implements forward-mode differentiation, and we use it by specifying where to evaluate \\(\bold{J}\\), and what values to use for \\(d\bold{x}\\)

```cpp
double x = 3.0;
double y = 2.0;
double dx = 1.0;
double dy = 1.0;
double df = __enzyme_fwddiff<double>((void*)f, enzyme_dup, x, dx, enzyme_dup, y, dy); 
printf("f(x) = %f, df = %f", f(x, y), df)
// prints f(x) = 6.500000, df = 4.750000
```

> But what if I only want to (forward) differentiate with respect to some of the inputs? 

There are two approaches:

1. Set `dx` or `dy` to zeroes as needed for unwanted derivatives

   ```cpp
   double dfdx = __enzyme_fwddiff<double>((void*)f, enzyme_dup, x, 1.0, enzyme_dup, y, 0.0); 
   double dfdy = __enzyme_fwddiff<double>((void*)f, enzyme_dup, x, 0.0, enzyme_dup, y, 1.0); 
   printf("dfdx = %f, dfdy = %f\n", dfdx, dfdy);
   // prints dfdx = 2.000000, dfdy = 2.750000
   ```

2. Specify `enzyme_const` to indicate which arguments are not to be differentiated
   ```cpp
   double dfdx = __enzyme_fwddiff<double>((void*)f, enzyme_dup, x, 1.0, enzyme_const, y); 
   double dfdy = __enzyme_fwddiff<double>((void*)f, enzyme_const, x, enzyme_dup, y, 1.0); 
   printf("dfdx = %f, dfdy = %f\n", dfdx, dfdy);
   // prints dfdx = 2.000000, dfdy = 2.750000
   ```

Option 1 has the benefit of flexibility-- we can choose to turn differentiation with respect to certain variables on or off at runtime. Option 2 hard codes which derivatives can be computed, which narrows scope and potentially improves performance. The appropriate choice will depend on the specific needs of your project.

Link to example: https://fwd.gymni.ch/OJMdAx

#### Reverse Mode

`__enzyme_autodiff` implements reverse-mode differentiation. We tell enzyme which function to differentiate, and pass information about where to evaluate the Jacobian. The `enzyme_out` specifier indicates which components (i.e. rows of the vector-jacobian product \\(\bold{J}^\top \lambda\\)) we want

```cpp
struct double2{ double x, y; };

...

double x = 3.0;
double y = 2.0;
auto [mu_x, mu_y] = __enzyme_autodiff<double2>((void*)f, enzyme_out, x, enzyme_out, y); 
printf("mu_x = %f, mu_y = %f\n", mu_x, mu_y);
// prints mu_x = 2.000000, mu_y = 2.750000
```

The value returned by `__enzyme_autodiff` is the concatenation of the different `enzyme_out` quantities. Here, we define a struct, `double2` to represent those output values (and use C++17 structured binding to split them into `mu_x, mu_y`).

> Note: it may be tempting to store the outputs in a `std::tuple< double, double >`, but the memory layout of `std::tuple` is implementation defined (e.g. some compilers implement `std::tuple<T, U, V>` with `V` first, `U` second, and `T` last). So, please don't store the outputs of `__enzyme_autodiff` in a `std::tuple`!

Link to example: https://fwd.gymni.ch/HMFTsY

## Free Functions: Return-By-Reference

Another possible way to implement the previous function is with an out-parameter:

```cpp
void f(double x, double y, double & output) { output = x * y + 1.0 / y; }
```

Differentiating functions like this with enzyme is similar to the return-by-value case, but with some small differences. 

If we only care about the derivative output, and not the function value itself, we can use the `enzyme_dupnoneed` descriptor. This lets the compiler optimize away unnecessary calculations associated with evaluating the output.

#### Forward Mode

```cpp
// these will be overwritten by __enzyme_fwddiff
double z = 0;
double dz = 0;
#if 1
__enzyme_fwddiff<void>((void*)f, enzyme_dup, x, dx, 
                                 enzyme_dup, y, dy, 
                                 enzyme_dup, &z, &dz);
printf("f(x,y) = %f, df = %f\n", z, dz);
// prints f(x,y) = 6.500000, df = 7.500000
#else
__enzyme_fwddiff<void>((void*)f, enzyme_dup, x, dx, 
                                 enzyme_dup, y, dy, 
                                 enzyme_dupnoneed, &z, &dz);
printf("f(x,y) = %f, df = %f\n", z, dz);
// prints f(x,y) = 0.000000, df = 7.500000
#endif
```

Note: the by-reference arguments of the function are passed to `__enzyme_fwddiff` by address. 

Link to example: https://fwd.gymni.ch/BtpzAA

#### Reverse Mode

vector-Jacobian product  \\(\boldsymbol{\mu} := \bold{J}^\top  \lambda\\\) is implemented as

```cpp
// lambda is an input to __enzyme_autodiff 
double lambda = 2.0;
#if 1
double2 mu = __enzyme_autodiff<double2>((void*)f, enzyme_out, x, 
                                                  enzyme_out, y, 
                                                  enzyme_dup, &z, &lambda); 
printf("z = %f, mu.x = %f, mu.y = %f\n", z, mu.x, mu.y);
// prints z = 6.500000, mu.x = 4.000000, mu.y = 5.500000
#else
double2 mu = __enzyme_autodiff<double2>((void*)f, enzyme_out, x, 
                                                  enzyme_out, y, 
                                                  enzyme_dupnoneed, &z, &lambda); 
printf("z = %f, mu.x = %f, mu.y = %f\n", z, mu.x, mu.y);
// prints z = 0.000000, mu.x = 4.000000, mu.y = 5.500000
#endif
```

Link to example: https://fwd.gymni.ch/eYmIzw


### Function Templates

Function templates are treated much the same way as regular functions, except
we need to explicitly include the template arguments when passing the function to
enzyme. For example, if we had the following definition:

```cpp
template < typename T >
void f(T x, T y, T & output) { output = x * y + 1.0 / y; }
```

Then the first argument looks like

```cpp
__enzyme_fwddiff<void>((void*)f<double>, enzyme_dup, x, dx, 
                                         enzyme_dup, y, dy, 
                                         enzyme_dupnoneed, &z, &dz);
```

Link to example: https://fwd.gymni.ch/WJVVRt


## Member Functions

Differentiating member functions with Enzyme is a little bit trickier, since a member
function in C++ is effectively a function with an implicitly passed argument (the `this` pointer).
This means that if we have an object with a member function

```cpp
struct MyObject {
    double f(double y) { return x * y + 1.0 / y; }
    double x;
};
```

we can't pass `&MyObject::f` directly to `__enzyme_fwddiff(...)`. Instead, what
we can do is write a free function that calls the desired member function:

```cpp
double wrapper(MyObject obj, double y) {
    return obj.f(y);
}
```

and pass _that_ to enzyme:

```cpp
double dfdy = __enzyme_fwddiff<double>((void*)wrapper, enzyme_const, obj, 
                                                       enzyme_dup, y, dy);
printf("dfdy = %f\n", dfdy);
// prints dfdy = 5.500000
```

A more general implementation of the wrapper function (that works with
different objects and argument types) is given below:

```cpp
template < typename T, typename ... arg_types >
auto wrapper(T obj, arg_types && ... args) {
    return obj.f(args ... );
}

...

double dfdy = __enzyme_fwddiff<double>((void*)wrapper<MyObject, double>, 
        enzyme_const, obj, 
        enzyme_dup, &y, &dy);
printf("dfdy = %f\n", dfdy);
// prints dfdy = 5.500000
```

> Question: why do `y` and `dy` now have `&` in front when being passed to `__enzyme_fwddiff`?

When passing information to `__enzyme_autodiff`, `__enzyme_fwddiff`:
- if the differentiated function takes an argument by value, then we pass it to enzyme by value
- if the differentiated function takes an argument by pointer, reference or rvalue-ref, then we pass it to enzyme by pointer

Link to example: https://fwd.gymni.ch/y0scib



If we want to differentiate with respect to the data members of the object (`x` in this case), we can annotate the object argument as `enzyme_dup`:

```cpp
MyObject dobj{2.0};
double dfdx = __enzyme_fwddiff<double>((void*)wrapper<MyObject, double>, 
        enzyme_dup, obj, dobj 
        enzyme_const, &y);
printf("dfdx = %f\n", dfdx);
```

Link to example: https://fwd.gymni.ch/wlC87F



## Functors and Lambda Functions

Functors are C++ objects that implement an `operator()` member, so they can be invoked like functions. Our previous example could have been implemented in the following way instead

```cpp
struct MyObject {
    double operator()(double y) const { return x * y + 1.0 / y; }
    double x;
};

double wrapper(const MyObject & f, double y) {  return f(y); }

...
    
double dfdy = __enzyme_fwddiff<double>((void*)(wrapper<MyObject, double>), 
    enzyme_const, (void*)&obj,
    enzyme_dup, &y, &dy);
printf("dfdy = %f\n", dfdy);
// prints dfdy = 0.750000
```

We can see that it is handled in the same way as regular member functions (passed to enzyme through a wrapper class).

--------

Lambda functions are another common C++ idiom for expressing function definitions in-line:

```cpp
auto f = [](double x, double y) { return x * y + 1.0 / y; };
```

Behind the scenes, the compiler expands the definitions of the lambdas above into functor objects

```cpp
// what the compiler "sees" when we type 
// auto f = [](double x, double y) { return x * y + 1.0 / y; };

class __lambda_f {
  public: 
    inline /*constexpr */ double operator()(double x, double y) const {
      return (x * y) + (1.0 / y);
    }
    
    using retType_f = double (*)(double, double);
    inline constexpr operator retType_f () const noexcept {
      return __invoke;
    };
    
    private: 
    static inline /*constexpr */ double __invoke(double x, double y) {
      return __lambda_f{}.operator()(x, y);
    }    
};

__lambda_f f = __lambda_f{};
```

This means there are a few ways to differentiate this kind of lambda function:

1. Passing directly to Enzyme

   ```cpp
   double dfdx = __enzyme_fwddiff<double>((void*)+f, 
     enzyme_dup, x, dx,
     enzyme_const, y);
   printf("dfdx = %f\n", dfdx);
   // dfdx = 6.200000
   ```

   Note: since `f` doesn't capture anything, it can implicitly convert to a function pointer, which Enzyme can handle directly. The weird `+f` in the first argument is a rarely-used unary operator that makes `f` convert to a function pointer.

2. Using the member function wrapper from above
   ```cpp
   double dfdy = __enzyme_fwddiff<double>((void*)(wrapper<decltype(f), double, double>), 
           enzyme_const, (void*)&f,
           enzyme_const, &x,
           enzyme_dup, &y, &dy);
   printf("dfdy = %f\n", dfdy);
   // dfdy = 0.750000
   ```

   Note: the lambda is explicitly cast to `(void*)` to suppress a compilation error. This error arises because, regularly, C++ has no way to specialize an `extern` template like `__enzyme_fwddiff(...)` using a type in a local scope (i.e. the lambda function), but Enzyme does not have this limitation.

3. Using a similar wrapper with non-type template parameter (requires C++20)

   ```cpp
   // C++20 or later
   template < auto obj, typename ... arg_types >
   auto wrapper(arg_types && ... args) {
       return obj(args ... );
   }
   
   ...
   
   double dfdy = __enzyme_fwddiff<double>((void*)(wrapper<f, double, double>), 
       enzyme_const, &x,
       enzyme_dup, &y, &dy);
   printf("dfdy = %f\n", dfdy);
   // dfdy = 0.750000
   ```



Link to examples: https://fwd.gymni.ch/wkgeoL

------

Instead, if a lambda function captures variables from its surrounding scope

```cpp
double x = 2.0;
auto f = [x](double y) { return x * y + 1.0 / y; };
```

then the compiler-generated functor object for the lambda expression is slightly different:

```cpp
// what the compiler "sees" when we type 
// double x = 2.0;
// auto f = [x](double y) { return x * y + 1.0 / y; };

class __lambda_f  {
 public: 
  inline /*constexpr */ double operator()(double y) const {
    return (x * y) + (1.0 / y);
  }
    
  __lambda_f(double & _x) : x{_x}{}
    
 private: 
  double x;
};

double x = 2.0;
__lambda_f f = __lambda_f{x};
```

An important difference is that by capturing, the lambda no longer generates the `static __invoke()` method or the implicit conversion to a function pointer, which means option #1 above (passing directly to Enzyme) will not work.

