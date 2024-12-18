# 🎨 Style Guide

## S.0 Code Guidelines

All guides follow the [C++ Core
Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

## S.1 Formatting

- Code shall follow libhal's
  [`.clang-format`](https://github.com/libhal/libhal/blob/main/.clang-format)
  file, which uses the Mozilla C++ style format as a base with some adjustments.
- Code shall follow libhal's
  [`.naming.style`](https://github.com/libhal/libhal/blob/main/.naming.style)
  file, which is very similar to the standard library naming convention:
  - CamelCase for template parameters.
  - CAP_CASE for MACROs (avoid MACROs in general).
  - lowercase snake_case for everything else.
  - prefix `p_` for function parameters.
  - prefix `m_` for private/protected class member.
- Refrain from variable names with abbreviations where it can be helped. `adc`,
  `pwm`, and `i2c` are extremely common so it is fine to leave them as
  abbreviations. Most people know the abbreviations more than the words that
  make them up. But words like `cnt` should be `count` and `cdl` and `cdh`
  should be written out as `clock_divider_low` and `clock_divider_high`.
  Registers do get a pass if they directly reflect the names in the data sheet
  which will make looking them up easier in the future.
- Use `#pragma once` as the include guard for headers.
- Every file must end with a newline character.
- Every line in a file must stay within a 80 character limit.
  - Exceptions to this rule are allowed. Use `// NOLINT` in these cases.
- Allowed number radix's for bit manipulation:
  - Only use binary (`0b1000'0011`) or hex (`0x0FF0`) for bit manipulation.
  - Never use decimal or octal as this is harder to reason about for most
    programmers.
- Every public API must be documented with the doxygen style comments (CI will
  ensure that every public API is documented fully).
- Include the C++ header version of C headers such as `<cstdint>` vs
  `<stdint.h>`.

## S.2 Refrain from performing manual bit set, clear, insertion, and extraction

Bit setting, clearing, and, more importantly, multi-bit insertion and multi-bit
extraction can be error prone and problematic. To solve this issue, we
recommend using `hal::bit_modify` or `hal::bit_value` from the `libhal-util`
library.

`hal::bit_modify` takes a reference to a volatile unsigned integer, copies its
value to a temporary variable, performs the operations on that temporary
variable, and, on destruction, assigns the temporary value to the referenced
volatile unsigned integer.

```C++
// For exposition:
reg_t* reg = get_peripheral_register();

hal::bit_modify(reg->control)
  .insert<pre_scalar>(freq() / desired_frequency)
  .clear<lower_power_mode>()
  .set<enable>();
```

`hal::bit_value` performs bit modifications on an integer. These are useful for
creating `constexpr` register values that already known at compile time, but
want to express the value with a combination of bit mask and modification.

```C++
constexpr auto default_pin_config = hal::bit_value(0U)
  .insert<mode>(0x04) // GPIO mode = 0x04
  .clear<high_slew_rate>()
  .set<high_speed_mode>()
  .to<std::uint32_t>();
```

Use `hal::bit_extract` from `libhal-util` library to perform bitwise extraction
operations.

```C++
while (hal::bit_extract<state>(reg->status) == states::busy) {
  continue;
}

// ... OR ...

float my_adc::driver_read() {
  constexpr auto capture_value = hal::bit_mask::from(4, 11);
  return hal::bit_extract<capture_value>(reg->status);
}
```

### S.2.1 Prefer named compile time bit-mask APIs

Always prefer giving names to bit masks as it makes it easier to understand
what its impact is on the hardware.

Always prefer using the APIs that take a bit masks as templates rather than an
input parameters unless the bit mask is not known at compile time. For a bit
mask to be used in a template means its `constexpr` or defined at compile time,
allowing the compiler more information about the operation, which it can use to
greatly optimize the operation.

```C++
constexpr auto state = hal::bit_mask::from(4, 5);
constexpr auto enable = hal::bit_mask::from(3);
constexpr auto prescale = hal::bit_mask::from(6, 13);
```

### S.2.1 Use temporary tail chaining when possible on `hal::bit_modify` and `hal::bit_value`

Temporary tail chaining looks like the following:

```C++
hal::bit_value(0U)
  .insert<hal::nibble_m<1, 2>>(7)
  .insert<hal::nibble_m<0>>(5)
  .to<std::uint32_t>();
```

Tail chaining allows you to all another API of the class from the return type.
GCC seems to perform better when it can see the whole lifetime of the object as
well as all of the operations performed on it. This tends to result in very
optimal codegen.

### S.2.2 [exception] Concatenation can use native syntax

In a lot of cases, using `hal::bit_modify` or `hal::bit_value` can be less
expressive than just using left and right shifts. So concatenating integers
together using `<<` and `|` is acceptable.

```C++
// auto == std::array<hal::byte, 2>
auto data = hal::write_then_read<2>(m_i2c, 0x11, addr, timeout);
// data[0] holds bits 4:11 and data[1] holds bits 0:3
std::uint32_t val = data[0] << 4 | data[1] >> 4;
return val;
```

vs

```C++
// auto == std::array<hal::byte, 2>
auto data = hal::write_then_read<2>(m_i2c, 0x11, addr, timeout);
auto value = hal::bit_value(0U)
  .insert<hal::nibble_m<1, 2>>(data[0])
  .insert<hal::nibble_m<0>>(hal::bit_extract<hal::nibble_m<1>>(data[1]))
  .to<std::uint32_t>();
return val;
```

Either of these is fine. Choose what you think is cleaner.

## S.3 Refrain from using MACROS

Only use macros if something cannot be done without using them. Usually macros
can be replaced with `constexpr` or const variables or function calls. A case
where macros are the only way is for HAL_CHECK() since there is no way
to automatically generate the boiler plate for returning if a function returns
and error in C++ and thus a macro is needed here to prevent possible mistakes
in writing out the boilerplate.

Only use preprocessor `#if` and the like if it is impossible to use
`if constexpr` to achieve the same behavior.

## S.4 Never include C++ `<iostream>` libraries

Applications incur an automatic 150kB space penalty for including any of the
ostream headers that also statically generate the global `std::cout` and the
like objects. This happens even if the application never uses any part of
`<iostream>` library. `<iostream>` can be used in libraries that will only be
used for host side testing.

## S.5 Refrain from memory allocations

Interfaces and drivers should refrain from APIs that force memory allocations
or implementations that allocate memory from heap. This means avoiding STL
libraries that allocate such as `std::string` or `std::vector`.

Many embedded system applications, especially the real time applications, do
not allow dynamic memory allocations. There are many reasons for this that can
be found MISRA C++ and AutoSAR.

## S.6 Drivers should not log to STDOUT or STDIN

Peripheral drivers must NOT log to stdout or stderr. This means no calls to

- `std::printf`
- `std::cout`
- `std::print` (C++26's version of print based on `std::format`)

Consider using the file I/O libraries in C, C++, python or some other language.
Would you, as a developer, ever imagine that opening, reading, writing, or
closing a file would (write?) to your console? Especially if there did not exist
a way to turn off logging. Most users would be very upset as this would not seem
like the role of the file I/O library to spam the console. This gets even worse
if a particular application has thousands of files and each operation is
logging.

The role of logging should be held by the application developer, not their
drivers or helper functions, unless the purpose of the helper functions or
driver is to write to console.

## S.7 Drivers should not purposefully halt OR terminate the application

Drivers are not entitled to halt the execution of the application and thus any
code block that would effectively end or halt the execution of the program
without giving control back to the application are prohibited.

As an example drivers should never call:

- `std::abort()`
- `std::exit()`
- `std::terminate()`
- any of their variants

This includes placing an **infinite loop block** in a driver.

An application should have control over how their application ends. A
driver should report severe errors to the application and let the application
decide the next steps. If a particular operation cannot be executed as intended,
then an appropriate `hal::exception` type should be thrown.

## S.8 Drivers should not pollute the global namespace

All drivers must be within the `hal` namespace or within their own bespoke
namespace.

Inclusion of a C header file full of register map structures is not allowed as
it pollutes the global namespace and tends to result in name collisions.

Care should be taken to ensure that the `hal` namespace is also as clean as
possible by placing structures, enums, const data, and any other symbols into
the driver's class's namespace like so:

```cpp
namespace hal::target
{
class target {
  struct register_map {
    std::uint32_t control1;
    std::uint32_t control2;
    std::uint32_t data;
    std::uint32_t status;
    // ..
  };

  struct control1_register {
    static constexpr auto channel_enable = hal::bit::range::from<0, 7>();
    static constexpr auto peripheral_enable = hal::bit::range::from<8>();
    // ...
  };

  // ...
};
}
```

## S.9 Interface should follow the public private API Scheme

See [private virtual method](http://www.gotw.ca/publications/mill18.htm)
for more details. Rationale can be found within that link as well.

## S.10 Avoid using `bool` as

### S.10.1 an object member

`bool` has very poor information density and takes up 8-bits per entry. If only
one `bool` is needed, then a bool is a fine object member. If multiple `bool`s
are needed, then use a `std::bitset` along with static `constexpr` index
positions in order to keep the density down to the lowest amount possible.

!!! note
    `std::bitset` seems to default to building blocks of size `int` meaning, on 32-bit architectures, this is 4-bytes for a bitset between 1 to 32.

### S.10.2 a parameter

See the article ["Clean code: The curse of a boolean
parameter"](https://medium.com/@amlcurran/clean-code-the-curse-of-a-boolean-parameter-c237a830b7a3)
for details as to why `bool` parameters are awful.

`bool` is fine if it is the only parameter and it acts as a lexical switch, for
example:

```c++
// This is fine because it reads as set "LED" voltage "level" to "FALSE"
led.level(false);
// This is fine because it reads as set "LED" voltage "level" to "TRUE"
led.level(true);
```

## S.11 Integrating third party libraries by source

In general, third party libraries should NOT be integrated into a library by
source. It should be depended upon using a package manager. But in some cases
third party libraries must be included by source. In these cases, the third
party libraries should be committed into a project, without modifications,
into the `include/<library_name>/third_party` directory. After that commit, the
third party libraries can be used by and integrated into the library code base,
in a following commit.

If a third party library is modified, that library must have a section at the
top of the file with the following description:

```C++
/**
 * [libhal] modifications to this file are as follows:
 *
 *    1. mod 1
 *    2. mod 2
 *    3. mod 3
 *    4. mod 4
 */

/**
 * <LICENSE GOES HERE!>
 */
```

Care must be taken to ensure that third party libraries do not conflict with
the licenses of libhal libraries and permit direct integration as well as
modification.

**Rationale:** Makes keeping track of changes and the history of files easier to
manage.

## S.12 Avoid `std::atomic`

Avoid using `std::atomic` in device libraries due to portability issues across
architectures. Device libraries are designed to work across architectures
meaning they cannot depend on platform specific constructs like this.

Note that `target` and `processor` libraries are allowed to use `std::atomic`
if it is available with their cross compiler and toolchain. In this case, the
we can know which target devices the software is running on, either the target
itself, which we already know can support it, or on a host machine for unit
testing, which is very likely to have a compiler that supports atomics.

## S.13 Avoid `<thread>`

Embedded system compilers tend to not provide an implementation of `<thread>`
because the choice of which threading model or multi-threading operating system
is left to the developer.

In general, `#include <thread>` will almost never work when cross compiling.

## S.14 Headers

Header files should be self-contained (compile on their own) and end in `.hpp`
for C++ files and `.h` for C files.

When a header declares inline functions or templates that clients of the header
will instantiate, the inline functions and templates must also have definitions
in the header. When all instantiations of a template occur in one .cc file,
either because they're explicit or because the definition is accessible to only
the .cc file, the template definition can be kept in that file.

### S.14.1 Include Guards

For ease of use, use `#pragma once` as your include guard. Usage of classic
include guards like:

```
#ifndef FOO_BAR_BAZ_H_
#define FOO_BAR_BAZ_H_

...

#endif  // FOO_BAR_BAZ_H_
```

Are annoying and error prone. Do not use these!

### S.14.2 Include What You Use

If a source or header file refers to a symbol defined elsewhere, the file
should directly include a header file which properly intends to provide a
declaration or definition of that symbol. It should not include header files
for any other reason.

Do not rely on transitive inclusions. This allows people to remove
no-longer-needed #include statements from their headers without breaking
clients. This also applies to related headers - foo.cc should include bar.h if
it uses a symbol from it even if foo.h includes bar.h.

### S.14.3 Include Order

Headers should be included in your headers and source files in the following
order:

- C standard library headers. Include using chevrons: `<>`.
- C 3rd party library packages. Include using chevrons: `<>`.
- C++ standard library headers. Include using chevrons: `<>`.
- C++ 3rd party Libraries' headers (`libhal`, `libhal-arm-mcu`, `libhal-util`,
  etc...). Include using chevrons: `<>`
- Local package/project headers. Include using quotes: `""`

For standard C headers use the C++ `<cstdio>` style over the C `<stdio.h>`
style.

Headers should be sorted alphabetically. `clang-format` will perform this work
for you. Rely on it vs doing it yourself.

Here is an example of how this should look:

```C++
// License at the top of the file with newline between the start of pragma once

#pragma once

// C headers first
#include <cstdio>
#include <cstdint>

// C library headers
#include <minimp3.h>

// C++ headers
#include <string_view>
#include <span>

// C++ Library headers
#include <libhal-util/output_pin.hpp>
#include <libhal-util/serial.hpp>
#include <libhal-util/steady_clock.hpp>
#include <libhal-util/stream_dac.hpp>

// Local Project
#include "resource_list.hpp"

// leave a blank line for the actual code
```

**Exception:** `boost.ut` must ALWAYS be the last include in the code in order
to allow `ostream` `operator<<` overloading to work.

## S.15 Classes

### S.15.1 Declaration Order

A class's visibility specifiers and member sections should appear in the
following order:

1. Public
2. Protected
3. Private

Omit any sections that would be empty.

Within each section, group similar declarations together and follow this order:

1. Types and type aliases:
    - Using directives (`using`)
    - Enum classes
    - Nested structs and classes
    - Friend classes and structs
2. Static constants
3. Factory functions (if applicable)
4. Constructors and assignment operators
5. Destructor
6. All other member functions (static and non-static member functions, as well
   as friend functions)
7. All other data members (static and non-static)

Do not put large method definitions inline within the class definition.
Typically, only trivial or performance-critical methods that are very short may
be defined inline. If the class is a template, then all functions must be
defined inline in the header file.

!!! note
    If a friend is a class or class function, then the friend should appear
    under the same visibility specifier as the friend. For example, if you are
    friending a private class function, then the friend function declaration
    should also appear in the private section of the friending class.

### S.15.2 Storing references

libhal drivers classes should not have reference member variables like so:

```C++
class my_driver {
public:
  my_driver(hal::adc16& p_adc): m_adc(p_adc) {
  }

private:
  hal::adc16& m_adc;  // ❌ Bad! Do not do this.
}
```

Reference members implicitly delete the copy and move constructors of a class
they are within because they themselves are not copyable. You cannot reassign
a reference after it is made.

Instead take the parameter as a reference but save its address as a pointer:

```C++
class my_driver {
public:
  my_driver(hal::adc16& p_adc): m_adc(&p_adc) {
  }

private:
  hal::adc16* m_adc = nullptr;  // ✅ Good!
}
```
