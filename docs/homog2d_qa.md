# Questions & Answers

[Manual main page](homog2d_manual.md)

<dl>
<dt>
Q: Is this library usable on a Windows platform?
</dt>
<dd>
A: Sure, should be!
I am not a Windows user myself, so I can't help on specific issues but as long as you are able to handle the inclusion of a file in your build process, it will work.
FWIW, the GH action CI test process includes building using the Microsoft Visual C++ compiler (known as MSVC),
[see here yamlfile](../.github/workflows/msvc.yml).
<br>
However, all the additional stuff here (Opencv graphical demo, test files, ...) will probably not build "out of the box", and will require some build tweaking.
But the library end-user doesn't need it.
</dd>

<dt>
Q: I am lost with all the "type" information, between the identifiers `Type`,  `type`,  `SType`,  `Dtype`, `FType`, ...
Could I have some clarification?
</dt>
<dd>

* `SType` is a static type that is redefined in all the "graphical" types.
It can be used to statically make sure that a call to a function meets some requirement on the parameter type.
* `Type` is an enum that also defines the nature of a given object, but it is object-based
(not type-based as `SType` is),
so it can be used to get the nature of an object, either with the free function
(`auto t = type(obj);`)
or with the corresponding member function.
(`auto t = obj.type();`).
You can get is as text with the free function `getString(Type)`.
* `Dtype` is an enum that is used to identify the numerical datatype of an object,
[see manual here](homog2d_manual.md#numtype).
It is the type returned by the `dtype()` member function that every geometrical object has.
You can have it as text with the `getString(Dtype)` free function.
* `FType` is a static type that holds for a type the underlying numerical type
(`float`, `double`, `long double` or other)
* `CommonType_` is just an typedef over the `std::variant` type holding the different geometrical primitives.
</dd>



<dt>
Q: How do I know the version that I have installed on my machine?
</dt>
<dd>
A: easy, add this in your app (or check for that symbol in the file).
<pre>
   std::cout << "version: " << HOMOG2D_VERSION << '\\n';
</pre>
</dd>

<dt>
Q: Why the choice of <tt>ttmath</tt> (https://www.ttmath.org/) as "big numbers" library?
</dt>
<dd>
A: Sure, other choices were possible. But it matched all these criterions:<br>
- C++ (GMP and MPFR have a C API)<br>
- header only (so really easy to install)<br>
- compatible licence (BSD)<br>
- used by other projects (Boost)<br>
- reasonnably easy API
</dd>

<a name="assert_trigger"></a>
<dt>
Q: I have an unexpected assert that gets triggered when I use this library in my code.
What shall I do?
</dt>
<dd>
A: Unfortunately, this can still happen at this time, and it is probably due to some numerical issue.
It is difficult to consider all the possible situations in test cases, so that one just got through.<br>
To get this solved,
please add <code>#define HOMOG2D_DEBUGMODE</code> just above <code>#include "homog2d.hpp"</code>,
and log the error stream of your program:<br>
<code>./myapp 2>stderr</code><br>
then open an issue (https://github.com/skramm/homog2d/issues) and include that output after removing non relevant parts, so this can be checked and corrected.
</dd>

<dt>
Q: I get a lot of warnings (`...may be used uninitialized in this function`) when running the test suite with GCC with `$ make test`, why?
</dt>
<dd>
A: This seems to be an issue with Catch, regular program shouldn't issue those, no worry
(this doesn't seem to happen with clang).
</dd>


<dt>
Q: Why do
<code>unionArea()</code> and <code>intersectArea()</code> not return the same data type?
</dt>
<dd>
A: because the first function returns a (flat) rectangle, and the second returns a polygon (as a closed <code>Polyline</code> object).
However, both can fail (if there is no common part of course), but you cannot return an invalid <code>FRect</code> object.
Thus the first function returns a type that can be checked for success.
A contrario, the second function will return a <code>Polyline</code> object, and in case of failure, it will just be empty, which is perfectly valid.
</dd>

<dt>
Q: What is the difference between the classes `rtp::Root` and `detail::Common` ?
Don't they serve the main purpose?
</dt>
<dd>
A: The class `detail::Common` is always enabled and serves as a common class for all the geometric types.
It is templated by the underlying numerical type, but does not provide polymorphic functions.
This is because a class templated cannot be virtual.

Thus, the need for the class `rtp::Root`
(only enabled if `HOMOG2D_ENABLE_PRTP` is defined, see
<a href="md_docs_homog2d_manual.html#build_options">build options</a>).
</dd>


<dt>
Q: `Circle_::center()` and `FRect_::getCenter()`?
Why not the same identifier?
</dt>
<dd>
A: because the first function returns a reference, and can be used to edit the value.
A contrario, the second function returns a value, and cannot be used to edit the value.
Thus, the intent is clearer.
</dd>


<dt>
Q: What is the difference between types `PointPair` and `Segment`?
Don't these two types just hold two points?
</dt>
<dd>
A: They both indeed hold two points, but they dont have the same semantic.
The latter represents a real geometrical object, thus cannot have a null length
(two identical points).
The first one just hold two arbitrary points.
</dd>


<dt>
Q: I notice the repo has a lot of branches. What's the point with all these branches?
</dt>
<dd>
A: Probably bad branch managment...
As "branching is free", I tend to create branches on the fly, but sometimes forget, and/or they are too different to sync up, so I just let them go.
But the point to remember is that "master" will always stay clean (build and tests ok), and will regularly be updated with bugfixes and new features.
</dd>


<dt>
Q: Any plans to deliver this as a (deb or rpm) package?
</dt>
<dd>
A: As the whole library is contained in a single file, there's not much difference between downloading  it or a package, so, no, not at present due to lack of time.
<br>
But if you feel like adding required code to build a package through a PR, you are welcome.
</dd>


<dt>
Q: Why does `getLmPoint( vector<Point2d> )` (and associated functions, [see here](homog2d_manual.md#extremum_points)) return a `std::pair`,
while the same functions applied on a Polyline (member of free functions) just return the point we are searching for?
</dt>
<dd>
A: Because when you are searching for a point in a container, there are some situations where not only you want the point but also its **position** in the container.
But with polylines (`CPolyline` or `OPolyline`), the concept of position makes no sense, as these are arbitrary and may change upon comparison or other events.
</dd>


<dt>
Q: Why Opencv as drawing backend?
</dt>
<dd>
-Short answer: because it works fine!
<br>
-Long(er) answer:
Besides the fact that I have already used it with other projects, it also has the great advantage of having all the drawing code for "advanced" primitives (circle, ellipse).
This doesn't seem to be the case for, say SDL2 (https://www.libsdl.org/) or other backends.
<br>
Alternatively, the library provides also SVG drawing.
</dd>

<dt>
Q: Why do yo use (Gnu)Make as build tool (and not CMake or another fancy thing)?
</dt>
<dd>
A: Because:<br>
 1. it is really powerful<br>
 2. there is no need for more<br>
 3. I like it

(Actually, I have the feeling that most people criticizing that tool never really understood how it works.)

Anyway, the library user doesn't need to use it:
as this is a "header-only" lib, he can use the tool he wants.
</dd>


</dl>

