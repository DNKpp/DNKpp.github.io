---
layout: post
title: Robust compile-error tests with CMake
subtitle: Seamless and automated CTest integration for compile-time diagnostics
gh-repo: DNKpp/mimicpp
gh-badge: [star, fork, follow]
tags: [CMake,C++,unit-test]
comments: true
readtime: true
mathjax: false
---

* Table of Contents
{:toc}

# Introduction

Last week, I had the pleasure of attending the **C++ Under The Sea conference** in Breda, NL.
Inspired by Björn Fahller’s excellent talk, *Using types to save your code's future*,
I found myself reflecting on an unresolved challenge in my mocking framework, *mimic++*.

Currently, *mimic++* makes extensive use of concept constraints throughout its interfaces.
This approach cleanly disables overloads when requirements aren’t met,
but it often leads to notoriously cryptic compiler error messages when no suitable overload is found.
While I appreciate the elegance of this style — it's central to template metaprogramming —
there are moments when users would benefit from much clearer explanations about what actually went wrong,
and — more importantly — how to fix them.

Within *mimic++*, users define *expectations* on *mocks* using an *expectation-builder*.
However, mistakes in these expectations can quickly lead to complex, hard-to-read error diagnostics.
During Björn’s talk, he demonstrated how to use `static_assert` not just for proper diagnostics,
but also to trim error messages down to their essentials.
Inspired by this, I went ahead and replaced several of my concept constraints with well-crafted `static_asserts`
— adding clear diagnostic messages that guide users directly to the relevant sections of the documentation.

Reworking the constraints proved straightforward.
But aiming for comprehensive test coverage raised a new question:
**How can I write unit tests for expected compile-time errors?**

This post recounts my exploration of that question and details the solution I implemented for *mimic++*.
While it doesn’t require deep C++ wizardry, it does demand a rough understanding of CMake and its target mechanics.

# Crafting Compile-Error Test-Cases

My goal was to keep each test case self-contained in a single source file,
laying out both the scenario and the specific error message fragments I expect.
Capturing the entire compiler output in a test would obviously be impractical and non-portable,
as error formats (and even wording) can vary across toolchains and platforms.
Instead, I decided to specify just those sub-strings that should be present within the compiler’s full error message.

To make this clear and easy to parse, I chose a simple convention:
each test case includes a multi-line comment with tagged boundaries for the expected error fragments.
```c++
/*
<begin-expected-compile-error>
...
<end-expected-compile-error>
*/
```
Each line between these tags represents a sub-string that must appear somewhere in the overall error output for the test to pass.
The actual code that will trigger the compile-error is highly case-specific and not of particular interest for this article.

The interesting question became:
how can we reliably extract these error expectations from the source file
— so they can be used programmatically within CMake?

# Extracting Error-Message Expectations with CMake

To automate the verification of compiler errors, the first step is to read the expected error strings from each test’s source file.
For this, I utilized CMake’s `file` command, which allows us to load the contents of the source file containing the test case.

The key is to programmatically identify the `<begin-expected-compile-error>` and `<end-expected-compile-error>` tags and extract the text between them.
To keep things organized, I encapsulated this logic in a dedicated function, `read_compile_error_definitions`, which takes two arguments:
the path to the source file, and the name of the variable where the extracted error definitions will be stored.
```cmake
function(read_compile_error_definitions SOURCE_FILE OUT_ERROR_DEFINITIONS)
    # ...
endfunction()
```

The function starts by reading the file content and ensuring it's not empty.
```cmake
file(READ ${SOURCE_FILE} FILE_CONTENT)
if (NOT FILE_CONTENT)
    message(FATAL_ERROR "Unable to read source-file: `${SOURCE_FILE}`")
endif ()
```

Next, we need to locate where in the file our error expectation block begins and ends.
For this, we search for the positions of `<begin-expected-compile-error>` and `<end-expected-compile-error>` within the full file content.
This step provides us with the raw index of both markers within the string (stored in the `DEF_BEGIN` and `DEF_END` variables).
```cmake
string(FIND "${FILE_CONTENT}" "<begin-expected-compile-error>" DEF_BEGIN)
string(FIND "${FILE_CONTENT}" "<end-expected-compile-error>" DEF_END)
```
Now that we know the position of the tags, we calculate the actual start offset for the error content (just after the `>` of the 'begin' tag)
and determine the exact length of the block to extract.
For this, we first compute the length of the begin-tag, then add it to its found index.
```cmake
set(BEGIN_TOKEN "<begin-expected-compile-error>")
string(FIND "${FILE_CONTENT}" "${BEGIN_TOKEN}" DEF_BEGIN)
string(LENGTH "${BEGIN_TOKEN}" BEGIN_TOKEN_LENGTH)
math(EXPR ERROR_START "${DEF_BEGIN} + ${BEGIN_TOKEN_LENGTH}")
```

The content's length is simply the difference between the start and end indices.
```cmake
string(FIND "${FILE_CONTENT}" "<end-expected-compile-error>" DEF_END)
math(EXPR ERROR_LENGTH "${DEF_END} - ${ERROR_START}")
```

With those values at hand, we can grab just the section of the file between the tags
(essentially the half-open range `[ERROR_START, ERROR_START + ERROR_LENGTH)`).
```cmake
string(SUBSTRING "${FILE_CONTENT}" ${ERROR_START} ${ERROR_LENGTH} ERROR_DEFINITIONS)
```

Are we done?
Not quite! Let’s look at an example to see why:
```c++
/*
<begin-expected-compile-error>
A first error sub-string
A second error sub-string
<end-expected-compile-error>
*/
```

Here, we actually want to treat each line between the tags as a separate sub-string to check for in the compiler output.
However, with our current approach, the extracted `ERROR_DEFINITIONS` variable ends up holding a single string like 
`\nA first error sub-string\nA second error sub-string\n`, which isn’t quite what we want.

Our first step is to remove any leading or trailing newlines or whitespace:
```cmake
string(STRIP "${ERROR_DEFINITIONS}" ERROR_DEFINITIONS)
```

At this point, `ERROR_DEFINITIONS` becomes `A first error sub-string\nA second error sub-string`, still joined by explicit newline characters.
The next step is to turn these error strings into a true CMake list so each sub-string is treated independently.
In CMake, lists are simply strings with elements separated by semicolons (`;`).
So, we need to replace all newline sequences with semicolons.

But to ensure our code works reliably across different platforms (which might use `\n`, `\r\n`, or `\r` line endings),
we use a regular expression to replace any sequence of `\r` and `\n` with a single semicolon:
```cmake
string(REGEX REPLACE "[\r\n]+" ";" ERROR_DEFINITIONS ${ERROR_DEFINITIONS})
```

This one-liner handles any combination or repetition of newline or carriage return characters, giving us a portable solution.

Finally, we need to pass this processed list back to the calling scope, so the extracted error sub-strings are available outside the function.
In CMake, this is achieved with the `PARENT_SCOPE` option.
Here, `OUT_ERROR_DEFINITIONS` acts like a pointer to the destination variable in the caller’s environment:
```cmake
set(${OUT_ERROR_DEFINITIONS} ${ERROR_DEFINITIONS} PARENT_SCOPE)
```

And that’s it! With these steps, we can robustly extract and prepare all expected error message fragments from the test case comments.

The full function now looks like this:
```cmake
function(read_compile_error_definitions SOURCE_FILE OUT_ERROR_DEFINITIONS)
    # Read in the content of the source-file and check that the content is not empty.
    file(READ ${SOURCE_FILE} FILE_CONTENT)
    if (NOT FILE_CONTENT)
        message(FATAL_ERROR "Unable to read source-file: `${SOURCE_FILE}`")
    endif ()
    
    # Find the begin of the content
    set(BEGIN_TOKEN "<begin-expected-compile-error>")
    string(FIND "${FILE_CONTENT}" "${BEGIN_TOKEN}" DEF_BEGIN)
    string(LENGTH "${BEGIN_TOKEN}" BEGIN_TOKEN_LENGTH)
    math(EXPR ERROR_START "${DEF_BEGIN} + ${BEGIN_TOKEN_LENGTH}")
    
    # Determine the length of the content
    string(FIND "${FILE_CONTENT}" "<end-expected-compile-error>" DEF_END)
    math(EXPR ERROR_LENGTH "${DEF_END} - ${ERROR_START}")
    
    # Extract the content and treat each line as it's own string
    string(SUBSTRING "${FILE_CONTENT}" ${ERROR_START} ${ERROR_LENGTH} ERROR_DEFINITIONS)
    string(STRIP "${ERROR_DEFINITIONS}" ERROR_DEFINITIONS)
    string(REGEX REPLACE "[\r\n]+" ";" ERROR_DEFINITIONS ${ERROR_DEFINITIONS})

    if (NOT ERROR_DEFINITIONS)
        message(FATAL_ERROR "Could not read compile-error definition(s) from `${SOURCE_FILE}`.")
    endif ()
    message(DEBUG "Expected compile-errors for case `${SOURCE_FILE}` are:\n\t`${ERROR_DEFINITIONS}`")

    # Mutate the destination variable in the caller's scope
    set(${OUT_ERROR_DEFINITIONS} ${ERROR_DEFINITIONS} PARENT_SCOPE)
endfunction()
```

# Registering Compile-Error Test-Cases

Now that we can extract the expected error fragments, the next step is to actually define our tests and hook them up to CTest for automated checking.

The goal is simple: for every compile-error test case source file, declare a unique test that attempts to build that file
and checks whether the expected error messages appear in the output.

First, I wanted the test name to be derived automatically from the actual source file’s name (minus its extension).
Fortunately, CMake makes this easy:
```cmake
cmake_path(GET SOURCE_FILE STEM NAME)
```

Next, each test case needs its own library target in CMake.
This target is given a unique name (by prefixing with something like `compile-error-test-`)
and includes only its corresponding source file.
Importantly, as the whole purpose is to provoke compile-errors, we need to make sure not compiling this target when building the project itself.
Therefore, we specify the `EXCLUDE_FROM_ALL` which then excludes this target from the general build.
```cmake
set(TARGET_NAME compile-error-test-${NAME})
add_library(${TARGET_NAME} EXCLUDE_FROM_ALL
    ${SOURCE_FILE}
)
```

The most interesting part is, of course, how we actually register the test.
CMake’s `CTest` module provides the `add_test` command, which is all we need (just remember to include `CTest` in your project).
```cmake
set(TEST_NAME compile-error-${NAME})
add_test(NAME ${TEST_NAME}
    COMMAND {CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target ${TARGET_NAME}
)
```
This is all it takes to register our compile-error test.
But notice — unlike traditional tests that run a compiled test-executable,
here we're simply triggering a build of the specific source file that's supposed to fail.

Let’s break it down:
- **`NAME ${TEST_NAME}`**  
    Sets the test name for CTest reporting.
- **`COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR}`**  
    This part constructs the build command that will be run by CTest:
    - `${CMAKE_COMMAND}` contains the absolute-path to the `cmake` command and thus just runs `cmake --build ${CMAKE_BINARY_DIR}` from a shell,
    - `${CMAKE_BINARY_DIR}` refers to the location where the CMake configuration is located.
- **`--target ${TARGET_NAME}`**  
    Restricts the build to the specific test target — your test case source file.

This setup ensures each compile-error test only attempts to compile its designated file, and the result is managed seamlessly within the CTest framework.

The current setup ensures that our compile-error test cases are built but we'll always end up in a test-failure,
since these builds are expected to produce compile-errors.
The final piece is to inform CTest that a build failure is anticipated
— and that the test should only pass if certain sub-strings (our expected error fragments) are present in the compiler output.

To achieve this, we can use the `PASS_REGULAR_EXPRESSION` property provided by CTest.
This allows us to specify that the test will be considered successful if all our extracted error sub-strings appear in the build output:
```cmake
read_compile_error_definitions(${SOURCE_FILE} EXPECTED_COMPILE_ERRORS)
set_tests_properties(${TEST_NAME} PROPERTIES
    PASS_REGULAR_EXPRESSION "${EXPECTED_COMPILE_ERRORS}"
)
```

{: .box-note}
**NOTE:** All error-strings are treated as regular expressions.  
Make sure to escape special characters (such as `.`, `*`, `?`, etc.) in your expectations as needed.  
Additionally, keep in mind that these regular expressions are interpreted by CMake, which uses its own regex dialect
— so double-check your patterns for CMake compatibility.

Revamping everything, our helper function for setting up a compile-error test now looks like this:
```cmake
function(check_compile_error SOURCE_FILE)
    # Extract the file-name (without extension) and generate the target- and test-name.
    cmake_path(GET SOURCE_FILE STEM NAME)
    set(TARGET_NAME compile-error-test-${NAME})
    set(TEST_NAME compile-error-${NAME})

    # Create a library-target which is excluded from the general build-process.
    add_library(${TARGET_NAME} EXCLUDE_FROM_ALL
        ${SOURCE_FILE}
    )

    # Register a test
    add_test(NAME ${TEST_NAME}
        COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target ${TARGET_NAME}
    )
    
    # This tells CTest, that we expect the build to fail with specific (sub-)messages.
    read_compile_error_definitions(${SOURCE_FILE} EXPECTED_COMPILE_ERRORS)
    set_tests_properties(${TEST_NAME} PROPERTIES
        PASS_REGULAR_EXPRESSION "${EXPECTED_COMPILE_ERRORS}"
    )
endfunction()
```

{: .box-note}
**NOTE:** This is just a minimal skeleton — there’s plenty of room to expand and tailor it to your needs.
As usual, you can add dependencies, set specific compile options, or customize your test environment further.

With this function in place, defining new compile-error test cases becomes trivial.
Just call `check_compile_error` for each desired test source file:
```cmake
check_compile_error("test1.cpp")
check_compile_error("test2.cpp")
# ...
```

Now, you can systematically test for and verify helpful compile-time error diagnostics in your CMake projects!

# Troubleshooting Windows

## Tackling Parallel-Test-Execution Issues

With the previously shown setup, running the compile-error tests for *mimic++* on Linux worked perfectly.
However, on Windows, the tests would occasionally fail with the following error:
```
error MSB3491: Could not write lines to file "x64\Debug\ZERO_CHECK\ZERO_CHECK.tlog\ZERO_CHECK.lastbuildstate".
The process cannot access the file '<build-dir>\x64\Debug\ZERO_CHECK\ZERO_CHECK.tlog\ZERO_CHECK.lastbuildstate' because it is being used by another process.
```

It turns out that MSVC employs a file-based locking mechanism, which prevents multiple builds from running in parallel in the same build directory.
Since I was using CTest with parallel execution enabled, this led to intermittent failures — a minor but real disappointment.

Fortunately, there's a straightforward fix.
CTest provides a synchronization feature via the `RESOURCE_LOCK` property.
By assigning this property to your test, you ensure that all compile-error related tests are run sequentially (never in parallel),
avoiding the file lock collisions entirely:
```cmake
if (MSVC)
    set_tests_properties(${TEST_NAME} PROPERTIES
        RESOURCE_LOCK compile-error-test-lock
    )
endif ()
```

By setting this property for each affected test, the parallel build issues on Windows are eliminated.

## Other Shenanigans

With CI/CD working happily, I encountered a different — and somewhat perplexing — problem when running CTest locally on my Windows laptop:
every test consistently failed due to the compiler not being able to find the standard header file `cstdint`.

This pointed to an underlying issue with how the compiler’s include directories were being set up for the test invocation.
On Windows builds often seem to depend on the `INCLUDE` environment variable being set.
If this variable is missing or incomplete, the compiler can't locate the standard library headers, and even basic includes like `cstdint` will fail.

The solution, with a helpful nudge from AI, was to ensure that the proper include paths are actively injected into the environment for each test invocation.
This is accomplished by extending the `add_test` call to use `cmake -E env`, which allows us to set environment variables for the invoked command.
By assigning `INCLUDE` to `${CMAKE_CXX_STANDARD_INCLUDE_DIRECTORIES}` (and appending any existing value),
we guarantee that the test's CMake-driven build process has access to all necessary standard headers:
```cmake
add_test(NAME ${TEST_NAME}
    COMMAND ${CMAKE_COMMAND} -E env "INCLUDE=${CMAKE_CXX_STANDARD_INCLUDE_DIRECTORIES};$ENV{INCLUDE}"
            ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target ${TARGET_NAME}
)
```

With this approach, the necessary include paths are always present, eliminating those frustrating header-not-found errors under Windows regardless of the local environment.

# Conclusion

I’ve explored a number of approaches for testing compile-time errors, but most solutions I found tend to rely on shell scripts and hand-written compile commands.

My goal was to achieve something simpler, neater, and more maintainable:
seamless integration right within my existing CMake and CTest-based testing workflow.
By leveraging only CMake logic, I was able to make tests cross-platform, and propagate all build settings, include paths,
and link options from the main project directly to the test case targets.
This harmony is especially satisfying: there’s no risk of “test drifting” from actual project builds or missing out on changes to build configuration.

What’s more, the actual implementation is elegant in its simplicity.
Every compile-error test is just a small C++ file with a documented expectation, no scripting fuss, no manual log checking.

Ultimately, this approach has not only enabled precise and robust compile-error testing, but has done so in a way that feels truly native to modern C++ project infrastructure
— and that, I find, is surprisingly satisfying!
