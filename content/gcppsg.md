#Google C++ Style Guide

##Header Files
In general, every .cc file should have an associated .h file. There are some common exceptions, such as unittests and small .cc files containing just a main() function.

###Self-contained Headers
Header files should be self-contained and end in .h. 

All header files should be self-contained. In other words, users and refactoring tools should not have to adhere to special conditions in order to include the header. Specifically, a header should have header guards, should include all other headers it needs, and should not require any particular symbols to be defined.

Textual inclusion: Examples are files that need to be included multiple times or platform-specific extensions that essencially are part of other headers. Such files should use the file extension .inc.

If a template or inline function is declared in a .h file, define it in that same file. As an exception, a function template that is explicitly instantiated for all relevant sets of template arguments, or that is a private member of a class, may be defined in the only .cc file that instantiates the template.

###The `#define` Guard
All header files should have `#define` guards to prevent multiple inclusion. The format of the symbol name should be *`<PROJECT>_<PATH>_<FILE>_H_`*.

eg: the file `foo/src/bar/baz.h` in project foo should have the following guard:

``
#ifndef FOO_BAR_BAZ_H_
#define FOO_BAR_BAZ_H_

...

#endif // FOO_BAR_BAZ_H_
``

###Forward Declarations
You may forward declare ordinary class in order to avoid unnecessary `#includes`.

**Definition:**

A "forward declaration" is a declaration of a class, funtion, or template without an associated definition. `#include` lines can often be replaced with forward declarations of whatever symbols are actually used by the client code.

**Pros:**

 - Unnecessary `#include` force the compiler to open more files and process more input.
 - They can also force your code to be recompiled more often, due to changes in the header.

**Cons:**

- It can be difficult to determine the correct form of a forward declaration in the presence of features like templates, typedefs, default parameteres, and using declarations.
- It can be difficult to determine whether a forward declaration or a full `#include` is needed for a given piece of code, particullarly when implicit conversion operations are involved. In extreme cases, replacing an `#include` with a forward declaration can silently change the meaning of code.
- Forward declaring multiple symbols from a header can be more verbose than simply `#includeing` the header.
- Forward declarations of functions and templates can prevent the header owners from making otherwise-compatible changes to their APIs; for example, widening a parameter type, or adding a template parameter with a default value.
- Forward declaring symbols from namespace `std::` usually yields undefined behavior.
- Structuring code to enable forward declarations (e.g. using pointer members instead of object members) can make the code slower and more complex.
- The practical efficiency benefits of forward declarations are unproven.

**Decisioins:**

- When using a function declared in a header file, always `#include` that header.
- When using a class template, prefer to `#include` its header file.
- When using an ordinary class, relying on a forward declarations is OK, but be wary of situations where a forward declaration may be insufficient or incorrect; when in doubt, just `#include` the appropriate header.
- Do not replace data members with pointers just to avoid an `#include`.

###Inline Functions
Define functions inline only when they are small, say, 10 lines or less.

**Definitions:**

You can declare functions in a way that allows the compiler to expand them inline rather than calling them through the usual function call mechanism.

**Pros:**

Inlining a function can generate more efficient object code, as long as the inlined function is small. Feel free to inline accessors and mutators, and other short, performance-critical functions.

**Cons:**

Overuse of inling can actually make programs slower. Depending on a function's size, inlining it can cause the code size to increase or decrease. Inlining a very small accessor function will usually decrease code size while inlining a very large function can dramatically increase code size. On modern processors smaller code usually runs faster due to better use of the instruction cache.

**Decision:**

A decent rule of thumb is to not inline a function if it is more than 10 lines long. Beware of destructors, when are often longer than they appear because of implicit member- and base-destructor calls!

Another useful rule of thumb: it's typically not cost effective to inline functions with loops or switch statements (unless, in the common case, the loop or switch statement is never executed).

It is important to know that functions are not always inlined even if they are declared as such; for example, virtual and recursive functions are not normally inlined. Usually recursive functions should not be inline. The main reason for making a virtual function inline is to place its definition in the class, either for convenience or to document its behavior, e.g., for accessors and mutators.

###Function Parameter Ordering
When defining a function, parameter order is: inputs, then outputs.

Parameters to C/C++ functions are either input to the function, output from the function, or both. Input parameters are usually values or const reference, while output and input/output parameters will be non-const pointers. When ordering function parameters, put all input-only parameters before any output parameters. In particular, do not add new parameters to the end of the function just because they are new; place new input-only parameters before the output parameters.

This is not a hard-and-fast rule. Parameters that are both input and output (often classes/structs) muddy the waters, and, as always, consistency with related functions may require you to bend the rule.

###Names and Orders of Includes
Use standard order for readability and to avoid hidden dependencies: Related header, C library, C++ library, other libraries' .h, your project's .h.

All of a project's header files should be listed as descendants of the project's source directory without use of UNIX directory shortcuts . (the current directory) or .. (the parent directory).

Within each section the includes should be ordered alphabetically. Note that older code might not conform to this rule and should be fixed when convenient.

You should include all the headers that defines the symbols you rely upon (except in cases of forward declaration). If you rely on symbols from `bar.h`, don't count on the fact that you included `foo.h` which included `bar.h`: include `bar.h` yourself, unless `foo.h` explicitly demonstrates its intent to provide you the symbols of `bar.h`. However, any includes present in the related header do not need to be included again in the related cc.

**Exception:**

Sometimes, system-specific code needs conditional includes. Such code can put conditional includes after other includes. Of course, keep your system-specific code small and localized. Example:

``
#include "foo/public/fooserver.h"

#include "base/port.h" // For LANG_CXX11.

#ifdef LANG_CXX11
#include <initializer_list>
#endif // LANG_CXX11
``