# Dev information

[Manual main page](homog2d_manual.md)

## 1 - Introduction

This page will gather misc. information useful for anyone wanting to contribute to code.
Some relevant details might be given here on how some functions are implemented.

To get the Doxygen-generated pages, you can run:
```
$ make doc
```
that will produce an "end-user" reference pages, thus ommitting some details (class private section).
To get the "full" reference, run:
```
$ make doc-dev
```

**2024/04/21**: A move to C++17 has been done.
This will greatly simplify the code, by enabling auto return types and replacing lots of SFINAE stuctures by `constexpr if`.
It also enables run-time polymorphism without pointers, with the help of `std::variant`
[details here](homog2d_manual.md#section_rtp).


## 2 - Git branches

`master` branch is supposed to stay stable, with all tests passing.
`devX` branches are where things happen.

Building tests, demos, showcases, ...: everything is handled through the makefile, using the "out-of-source" principle:
everything that gets build ends up in the `BUILD` folder

## 3 - Makefile usage

### 3.1 - Available targets

* (default): same as `test`
* `clean`: erase everything that was built (in the `BUILD` folder)
* `test`: builds and runs the unit test with and without the `HOMOG2D_OPTIMIZE_SPEED` build option
* `testall`: builds and runs the unit test for the 3 different numerical types
* `check`: runs cppcheck (static analysis) ( https://cppcheck.sourceforge.io/ )
* `install`: `cp homog2d.hpp /usr/local/include`
* `doc`: build html reference (requires doxygen), with only public user API
* `doc-dev`: build full html reference (requires doxygen), including private class members
* `nobuild`: checks that the files in `misc/no_build` contain illegal code (part of the test process)


**Targets only available if Opencv is installed:**

* `demo`: build and run the Opencv interactive demo
* `showcase`: build and run the gif figures used in manual
* `test-fig`: build and run the code that generates the figures used in test cases
* `doc-fig`: build and run the code used to produce the figures of the manual

**Targets only available if LaTeX installed:**

* `doc-fig-tex`: build the LaTeX figures of the manual (TikZ based)

### 3.2 - Available options

These can be passed to make:
```
$ make <target> <option=Y|N>
```

* `USE_OPENCV`: enables the OpenCv additional features (useful for "test" targets)
* `USE_TINYXML2`: enables the SVG import (through Tinyxml2) additional features (useful for "test" targets)
* `USE_EIGEN`: enables the Eigen3 additional features
* `DEBUG`:  adds `-g` flag to compiler options

### 3.3 - Additional details

The `test-fig` target generates the images of some of the code used in the test cases.
This is solely to be able to see what is actually tested.
This code is located in `misc/figures_test`.
A valid program generating the image is generated by pasting together the considered file and `misc/figures_test/t_header.cxx` and one of the footer files
`misc/figures_test/t_footer_*.cpp`.
The makefile then compiles it and runs it.

## 4 - Code details

### 4.1 - Partial template implementation tricks

To be able to templatize all the code on the root numerical data type (float, double, ...), we implement some trick.
As the `LPBase` class is already templatized on the type (`typ::IsPoint` or `typ::IsLine`),
it would require a partial template specialization to define the behavior of each member function (or free function),
depending on the basic type (Line or Point), and still templatize on the numerical type.
C++ does not allow this.

Thus, the trick here is to call in each function a "sub" private function (prefixed with `impl_`) that gets overloaded by the datatype (point or line).
To achieve this overloading, each of these functions receives as additional (dummy) argument an object of type `BaseHelper`, templated by the numerical type.
In the definition of the function, this additional argument value is ignored,
it is there just so that the compiler can select the correct overload.

The different implementations are written as two `impl_` private functions that are templated by the numerical data type.
If the situation only makes sense for one of the types (for example `getAngle()` cannot be considered for two points), then
the implementation of that type only holds a `static_assert`, so that error can be catched at build time.

This is mostly used with the three "base" classes, located in namespace `base`:
- `base::LPBase`, specialized as `Point2d_` and `Line2d_`
- `base::PolylineBase`specialized as `CPolyline_` and `OPolyline`
- `base::SegVec`, specialized as `Segment_` and `OSegment_`

### 4.2 - Common classes and polymorphism

With C++, polymorphism can be achieved in two ways:

- classical runtime polymorphism, achieved with virtual functions.
This is particularly useful when the need is to have a container holding objects of different types, and then calling a virtual member function on each object.
- static Compile-time polymorphism, using function overloading.

The general design here is to use compile-time polymorphism.
However, in some situations, runtime polymorphism is required.
For example when one wants to import an SVG file, because we can't determine the types of objects that will be read.

When inheriting concrete classes from a class template, we face a problem if we need to achieve runtime polymorphism,
Consider this inheritance:
```
template<typename T>
struct BaseClass
{
	virtual void foo() = 0;
};
template<typename T>
struct ConcreteClassA
{
	void foo() { ... }
};
template<typename T>
struct ConcreteClassB
{
	void foo() { ... }
};

```

But now, if we want to build a vector holding pointers on concrete objects of different types, then we face a problem:
we cannot define a common base type !:
```
std::vector<BaseClass<???>*> vec;
```

So, first, why do we need a base class template in the beginning?
Why not use a non-templated class?

The answer is two-parts:

- first, an non-templated base class holding pure (or not pure) virtual function will add an overhead, because of the vtable that gets created, and this is not always needed.
- second, in the present situation, using a class template enables us to have some common functionnality on all concrete types.
Here, the member function `Dtype dtype()` enables us to fetch the underlying numerical type on all inherited concrete types
(see [build options](homog2d_manual.md#numtype)).

This is why we have here a double inheritance pattern, on all concrete types (geometric primitives):

- The class template `Common<T>`, provides common stuff for all types, and holds a default implementation for member functions not defined in the concrete types.
- The non-templated class `Root`, provides real-time polymorphism through pointers.
This latter inheritance is only enabled if symbol `HOMOG2D_ENABLE_PRTP` is defined
(see [build options](homog2d_manual.md#build_options)).

However, the pointer-based technique is error prone: no type safety, and bad casting will lead to segfaults.
So we have also another way of handling runtime polymorphism, by switching to C++17 and using `std::variant`.
[See here](homog2d_manual.md#section_rtp) for details.

### 4.3 - Bounding Boxes

The concept of bounding box is easy to understand for the some primitives
(classes `FRect`, `Circle`, `Ellipse`),
and it is validated with the trait class template `HasBB<>`.
For Segments, it may throw, if the points share some common coordinate (thus not having an area).
For the two "polylines" classes (`OPolyline` and `CPolyline`), it **may** exist (at compile time), but not always succeed.
Indeed, these two types can have only two points, thus being equivalent to segments, that do not have a bounding box
(consider vertical or horizontal segments).

On other primitives, the concept of bounding box makes no sense:

* For lines, as that are infinite, a bounding box can not be defined.
* For points, as a bounding box is implemented as a `FRect` object and that this type cannot have a null area, it can not be defined either.
* For segments, as these may not have an area, no `getBB()` member function is enabled.

The table below summarizes what happens when attempt to call `getBB()` on an object:

|     Type    |               |
|-------------|---------------|
| `Point2d`   | no build      |
| `Line2d`    | no build      |
| `Segment`   | no build      |
| `OSegment`  | no build      |
| `Circle`    | never throws  |
| `FRect`     | never throws  |
| `OPolyline` | may throw     |
| `CPolyline` | may throw     |
| `Ellipse`   | never throws  |

for `OPolyline` and `CPolyline`, the function will throw either if those primitives are empty, or if the points they hold do not define an area.

**Bounding box of two objects**

This library provides a free function `getBB( t1, t2 )` that will return the bounding boxes holding `t1` and `t2`, whatever the types involved.

* if one of the two objects is a line, no bounding box can be computed, so a call to `getBB()` with two lines will not compile.
* if both of the arguments have an area, then there is no problem.
* if one of the arguments is a point or a segment, then a common bounding box may exist, but the function will throw if no area can be defined
(say, for two points having a common coordinate, or two colinear segments).
* for Polyline types (open or closed, makes no difference), it depends:
  * if both are empty, then no common area can de defined, and the function will throw an error.
  * if one of them is empty, we attemps go get the bounding box for the other (will throw if not exist)
  * if both are not empty, we attemps go get the common bounding box, but this can fail if they are colinear


## 5 - Testing
<a name="testing"></a>

A lot of unit-testing is included in this package.
It uses the Catch framework, version 2 (header-only version).

Unit-testing is automatically done with Github Actions.
Two Yaml workflow files are present.
- one [msvc.yml](.github/workflows/msvc.yml) is only dedicated to checking that the Microsoft C++ compiler is able to build the software.
- the [other (c-cpp.yml)](.github/workflows/c-cpp.yml) runs the unit test file using both GCC and Clang, on two different platforms (Ubuntu 22 and Ubuntu 24).

If both of these workflows succeed, it will generate the badge on front page.

Once the test app has been build (with `$ make test`), you can run a single test with:
```
$ BUILD/<testappname> [tagname]
```

To get the list of available tests, you can check the test file, or run:
```
$ make test-list
```

The "test" target actually call 3 targets:
- the "test2" target (see below);
- the "no-build" target, that make sure that what shouldn't build... does not;
- the runtime polymorphism tests (target `test-rtp`)

The "test2" target runs the unit test file with 4 different combinations of configurations (see [build options](homog2d_manual.md#build_options)),
identified by a letter followed by `Y` (yes) or 'N' (no).
The configurations tested are:
- `S`: with or without the speed optimisations, relevant for ellipses (symbol `HOMOG2D_OPTIMIZE_SPEED`)
- `V`: with or without the variant based runtime polymorphism (symbol `HOMOG2D_ENABLE_VRTP`)

So this defines 4 targets/programs, named `test_SYVY`, `test_SYVN`, `test_SNVY`, `test_SNVN`, that are run by that target

This "test2" target also runs "test3", that only checks that everything build fine when using the library in a single .cpp program_invocation_name
or when using it in a multiple file app.
This is actually what is tested when using the Microsoft C++ compiler (see [corresponding workflow file](../.github/workflows/msvc.yml)).


## 6 - Coding style

- TABS, not spaces (1 byte per level)
- types have first character uppercase, variables and functions are lowercase
- `camelCase` for identifiers
- class member variables are prefixed with an underscore (`_`)
- spaces after parenthesis (`if( someBool )`)
- private and protected member function are prefixed with `p_` (or `impl_` for tag dispatch implementation)
- all symbols start with `HOMOG2D_`, to avoid name collisions


## 7 - Big Numbers support
<a name="ttmath_devinfo"></a>

To enable the usage of the `ttmath` library ([see here](homog2d_manual.md#bignum)), some edits had the be done on the code,
because this library provides its own maths functions.
The problem is that they do not have the same name as in the standard library.
For example the `sin()` function is named `Sin()` in the `ttmath` library (as opposed to `std::sin()`).

One solution to handle this would have been to create a sub-namespace `num` that would hold these math functions:
```
namespace num {
	template<typename FP>
	HOMOG2D_INUMTYPE sin( FP val ) const
	{
		return impl_sin( val, detail::DataFpType<FPT>() );
	}
} // namespace num
```
And these would then call a hidden implementation, specialised using a dummy argument on either a standard numerical type or a ttmath type
(**update 2025/C++17: or use a `if constepr`**).

But this seemed a bit "over engineered", and a simpler solution was choosen, using macros.
As it is admitted that the standard types are no longer usable when `HOMOG2D_USE_TTMATH` is defined, a simple text replacement is used:
In the library code, all the maths functions are prefixed with `homog2d_` (for example `homog2d_asin()` for the inverse sinus function).
Depending if the symbol `HOMOG2D_USE_TTMATH` is defined or not, these symbols are replaced with the relevant string.

For example `homog2d_asin` will be replaced by `ttmath::ASin` if `HOMOG2D_USE_TTMATH` is defined, and by `std::asin` if not:

```
#ifdef HOMOG2D_USE_TTMATH
	#define homog2d_sin ttmath::Sin
#else
	#define homog2d_sin std::sin
#endif
```

## 8 - Traits classes

The namespaces `trait` holds several traits classes that are used through out the code.
Two of these, `HasBB` ("Has Bounding Box") and `HasArea`, are detailed in the table below, showing their "constexpr" boolean value function of the given type:

|   Type    | HasBB | HasArea |
|-----------|-------|---------|
| Point2d   | false |  false  |
| Line2d    | false |  false  |
| Segment   | false |  false  |
| OSegment  | false |  false  |
| FRect     | true  |  true   |
| Circle    | true  |  true   |
| Ellipse   | true  |  true   |
| OPolyline | true  |  false  |
| CPolyline | true  |  true   |


