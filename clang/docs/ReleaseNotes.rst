===========================================
Clang |release| |ReleaseNotesTitle|
===========================================

.. contents::
   :local:
   :depth: 2

Written by the `LLVM Team <https://llvm.org/>`_

.. only:: PreRelease

  .. warning::
     These are in-progress notes for the upcoming Clang |version| release.
     Release notes for previous releases can be found on
     `the Releases Page <https://llvm.org/releases/>`_.

Introduction
============

This document contains the release notes for the Clang C/C++/Objective-C
frontend, part of the LLVM Compiler Infrastructure, release |release|. Here we
describe the status of Clang in some detail, including major
improvements from the previous release and new feature work. For the
general LLVM release notes, see `the LLVM
documentation <https://llvm.org/docs/ReleaseNotes.html>`_. For the libc++ release notes,
see `this page <https://libcxx.llvm.org/ReleaseNotes.html>`_. All LLVM releases
may be downloaded from the `LLVM releases web site <https://llvm.org/releases/>`_.

For more information about Clang or LLVM, including information about the
latest release, please see the `Clang Web Site <https://clang.llvm.org>`_ or the
`LLVM Web Site <https://llvm.org>`_.

Potentially Breaking Changes
============================
These changes are ones which we think may surprise users when upgrading to
Clang |release| because of the opportunity they pose for disruption to existing
code bases.


C/C++ Language Potentially Breaking Changes
-------------------------------------------

- The default extension name for PCH generation (``-c -xc-header`` and ``-c
  -xc++-header``) is now ``.pch`` instead of ``.gch``.
- ``-include a.h`` probing ``a.h.gch`` is deprecated. Change the extension name
  to ``.pch`` or use ``-include-pch a.h.gch``.

C++ Specific Potentially Breaking Changes
-----------------------------------------
- The name mangling rules for function templates has been changed to take into
  account the possibility that functions could be overloaded on their template
  parameter lists or requires-clauses. This causes mangled names to change for
  function templates in the following cases:

  - When a template parameter in a function template depends on a previous
    template parameter, such as ``template<typename T, T V> void f()``.
  - When the function has any constraints, whether from constrained template
      parameters or requires-clauses.
  - When the template parameter list includes a deduced type -- either
      ``auto``, ``decltype(auto)``, or a deduced class template specialization
      type.
  - When a template template parameter is given a template template argument
      that has a different template parameter list.

  This fixes a number of issues where valid programs would be rejected due to
  mangling collisions, or would in some cases be silently miscompiled. Clang
  will use the old manglings if ``-fclang-abi-compat=17`` or lower is
  specified.
  (`#48216 <https://github.com/llvm/llvm-project/issues/48216>`_),
  (`#49884 <https://github.com/llvm/llvm-project/issues/49884>`_), and
  (`#61273 <https://github.com/llvm/llvm-project/issues/61273>`_)

ABI Changes in This Version
---------------------------
- Following the SystemV ABI for x86-64, ``__int128`` arguments will no longer
  be split between a register and a stack slot.

What's New in Clang |release|?
==============================
Some of the major new features and improvements to Clang are listed
here. Generic improvements to Clang as a whole or to its underlying
infrastructure are described first, followed by language-specific
sections with improvements to Clang's support for those languages.

C++ Language Changes
--------------------

C++20 Feature Support
^^^^^^^^^^^^^^^^^^^^^

C++23 Feature Support
^^^^^^^^^^^^^^^^^^^^^

C++2c Feature Support
^^^^^^^^^^^^^^^^^^^^^

- Implemented `P2169R4: A nice placeholder with no name <https://wg21.link/P2169R4>`_. This allows using ``_``
  as a variable name multiple times in the same scope and is supported in all C++ language modes as an extension.
  An extension warning is produced when multiple variables are introduced by ``_`` in the same scope.
  Unused warnings are no longer produced for variables named ``_``.
  Currently, inspecting placeholders variables in a debugger when more than one are declared in the same scope
  is not supported.

  .. code-block:: cpp

    struct S {
      int _, _; // Was invalid, now OK
    };
    void func() {
      int _, _; // Was invalid, now OK
    }
    void other() {
      int _; // Previously diagnosed under -Wunused, no longer diagnosed
    }

- Attributes now expect unevaluated strings in attributes parameters that are string literals.
  This is applied to both C++ standard attributes, and other attributes supported by Clang.
  This completes the implementation of `P2361R6 Unevaluated Strings <https://wg21.link/P2361R6>`_


Resolutions to C++ Defect Reports
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

C Language Changes
------------------
- ``structs``, ``unions``, and ``arrays`` that are const may now be used as
  constant expressions.  This change is more consistent with the behavior of
  GCC.

C23 Feature Support
^^^^^^^^^^^^^^^^^^^
- Clang now accepts ``-std=c23`` and ``-std=gnu23`` as language standard modes,
  and the ``__STDC_VERSION__`` macro now expands to ``202311L`` instead of its
  previous placeholder value. Clang continues to accept ``-std=c2x`` and
  ``-std=gnu2x`` as aliases for C23 and GNU C23, respectively.
- Clang now supports `requires c23` for module maps.

Non-comprehensive list of changes in this release
-------------------------------------------------

New Compiler Flags
------------------

* ``-fverify-intermediate-code`` and its complement ``-fno-verify-intermediate-code``.
  Enables or disables verification of the generated LLVM IR.
  Users can pass this to turn on extra verification to catch certain types of
  compiler bugs at the cost of extra compile time.
  Since enabling the verifier adds a non-trivial cost of a few percent impact on
  build times, it's disabled by default, unless your LLVM distribution itself is
  compiled with runtime checks enabled.
* ``-fkeep-system-includes`` modifies the behavior of the ``-E`` option,
  preserving ``#include`` directives for "system" headers instead of copying
  the preprocessed text to the output. This can greatly reduce the size of the
  preprocessed output, which can be helpful when trying to reduce a test case.
* ``-fassume-nothrow-exception-dtor`` is added to assume that the destructor of
  an thrown exception object will not throw. The generated code for catch
  handlers will be smaller. A throw expression of a type with a
  potentially-throwing destructor will lead to an error.

* ``-fopenacc`` was added as a part of the effort to support OpenACC in clang.

Deprecated Compiler Flags
-------------------------

Modified Compiler Flags
-----------------------

* ``-Woverriding-t-option`` is renamed to ``-Woverriding-option``.
* ``-Winterrupt-service-routine`` is renamed to ``-Wexcessive-regsave`` as a generalization

Removed Compiler Flags
-------------------------

* ``-enable-trivial-auto-var-init-zero-knowing-it-will-be-removed-from-clang`` has been removed.
  It has not been needed to enable ``-ftrivial-auto-var-init=zero`` since Clang 16.

Attribute Changes in Clang
--------------------------
- On X86, a warning is now emitted if a function with ``__attribute__((no_caller_saved_registers))``
  calls a function without ``__attribute__((no_caller_saved_registers))``, and is not compiled with
  ``-mgeneral-regs-only``
- On X86, a function with ``__attribute__((interrupt))`` can now call a function without
  ``__attribute__((no_caller_saved_registers))`` provided that it is compiled with ``-mgeneral-regs-only``

- When a non-variadic function is decorated with the ``format`` attribute,
  Clang now checks that the format string would match the function's parameters'
  types after default argument promotion. As a result, it's no longer an
  automatic diagnostic to use parameters of types that the format style
  supports but that are never the result of default argument promotion, such as
  ``float``. (`#59824: <https://github.com/llvm/llvm-project/issues/59824>`_)

Improvements to Clang's diagnostics
-----------------------------------
- Clang constexpr evaluator now prints template arguments when displaying
  template-specialization function calls.
- Clang contexpr evaluator now displays notes as well as an error when a constructor
  of a base class is not called in the constructor of its derived class.
- Clang no longer emits ``-Wmissing-variable-declarations`` for variables declared
  with the ``register`` storage class.
- Clang's ``-Wtautological-negation-compare`` flag now diagnoses logical
  tautologies like ``x && !x`` and ``!x || x`` in expressions. This also
  makes ``-Winfinite-recursion`` diagnose more cases.
  (`#56035: <https://github.com/llvm/llvm-project/issues/56035>`_).
- Clang constexpr evaluator now diagnoses compound assignment operators against
  uninitialized variables as a read of uninitialized object.
  (`#51536 <https://github.com/llvm/llvm-project/issues/51536>`_)
- Clang's ``-Wformat-truncation`` now diagnoses ``snprintf`` call that is known to
  result in string truncation.
  (`#64871: <https://github.com/llvm/llvm-project/issues/64871>`_).
  Existing warnings that similarly warn about the overflow in ``sprintf``
  now falls under its own warning group ```-Wformat-overflow`` so that it can
  be disabled separately from ``Wfortify-source``.
  These two new warning groups have subgroups ``-Wformat-truncation-non-kprintf``
  and ``-Wformat-overflow-non-kprintf``, respectively. These subgroups are used when
  the format string contains ``%p`` format specifier.
  Because Linux kernel's codebase has format extensions for ``%p``, kernel developers
  are encouraged to disable these two subgroups by setting ``-Wno-format-truncation-non-kprintf``
  and ``-Wno-format-overflow-non-kprintf`` in order to avoid false positives on
  the kernel codebase.
  Also clang no longer emits false positive warnings about the output length of
  ``%g`` format specifier and about ``%o, %x, %X`` with ``#`` flag.
- Clang now emits ``-Wcast-qual`` for functional-style cast expressions.
- Clang no longer emits irrelevant notes about unsatisfied constraint expressions
  on the left-hand side of ``||`` when the right-hand side constraint is satisfied.
  (`#54678: <https://github.com/llvm/llvm-project/issues/54678>`_).
- Clang now prints its 'note' diagnostic in cyan instead of black, to be more compatible
  with terminals with dark background colors. This is also more consistent with GCC.
- Clang now displays an improved diagnostic and a note when a defaulted special
  member is marked ``constexpr`` in a class with a virtual base class
  (`#64843: <https://github.com/llvm/llvm-project/issues/64843>`_).
- ``-Wfixed-enum-extension`` and ``-Wmicrosoft-fixed-enum`` diagnostics are no longer
  emitted when building as C23, since C23 standardizes support for enums with a
  fixed underlying type.
- When describing the failure of static assertion of `==` expression, clang prints the integer
  representation of the value as well as its character representation when
  the user-provided expression is of character type. If the character is
  non-printable, clang now shows the escpaed character.
  Clang also prints multi-byte characters if the user-provided expression
  is of multi-byte character type.

  *Example Code*:

  .. code-block:: c++

     static_assert("A\n"[1] == U'🌍');

  *BEFORE*:

  .. code-block:: text

    source:1:15: error: static assertion failed due to requirement '"A\n"[1] == U'\U0001f30d''
    1 | static_assert("A\n"[1] == U'🌍');
      |               ^~~~~~~~~~~~~~~~~
    source:1:24: note: expression evaluates to ''
    ' == 127757'
    1 | static_assert("A\n"[1] == U'🌍');
      |               ~~~~~~~~~^~~~~~~~

  *AFTER*:

  .. code-block:: text

    source:1:15: error: static assertion failed due to requirement '"A\n"[1] == U'\U0001f30d''
    1 | static_assert("A\n"[1] == U'🌍');
      |               ^~~~~~~~~~~~~~~~~
    source:1:24: note: expression evaluates to ''\n' (0x0A, 10) == U'🌍' (0x1F30D, 127757)'
    1 | static_assert("A\n"[1] == U'🌍');
      |               ~~~~~~~~~^~~~~~~~
- Clang now always diagnoses when using non-standard layout types in ``offsetof`` .
  (`#64619: <https://github.com/llvm/llvm-project/issues/64619>`_)
- Clang now diagnoses redefined defaulted constructor when redefined
  defaulted constructor with different exception specs.
  (`#69094: <https://github.com/llvm/llvm-project/issues/69094>`_)
- Clang now diagnoses use of variable-length arrays in C++ by default (and
  under ``-Wall`` in GNU++ mode). This is an extension supported by Clang and
  GCC, but is very easy to accidentally use without realizing it's a
  nonportable construct that has different semantics from a constant-sized
  array. (`#62836 <https://github.com/llvm/llvm-project/issues/62836>`_)

- Clang changed the order in which it displays candidate functions on overloading failures.
  Previously, Clang used definition of ordering from the C++ Standard. The order defined in
  the Standard is partial and is not suited for sorting. Instead, Clang now uses a strict
  order that still attempts to push more relevant functions to the top by comparing their
  corresponding conversions. In some cases, this results in better order. E.g., for the
  following code

  .. code-block:: cpp

      struct Foo {
        operator int();
        operator const char*();
      };

      void test() { Foo() - Foo(); }

  Clang now produces a list with two most relevant builtin operators at the top,
  i.e. ``operator-(int, int)`` and ``operator-(const char*, const char*)``.
  Previously ``operator-(const char*, const char*)`` was the first element,
  but ``operator-(int, int)`` was only the 13th element in the output.
  However, new implementation does not take into account some aspects of
  C++ semantics, e.g. which function template is more specialized. This
  can sometimes lead to worse ordering.


- When describing a warning/error in a function-style type conversion Clang underlines only until
  the end of the expression we convert from. Now Clang underlines until the closing parenthesis.

  Before:

  .. code-block:: text

    warning: cast from 'long (*)(const int &)' to 'decltype(fun_ptr)' (aka 'long (*)(int &)') converts to incompatible function type [-Wcast-function-type-strict]
    24 | return decltype(fun_ptr)( f_ptr /*comment*/);
       |        ^~~~~~~~~~~~~~~~~~~~~~~~

  After:

  .. code-block:: text

    warning: cast from 'long (*)(const int &)' to 'decltype(fun_ptr)' (aka 'long (*)(int &)') converts to incompatible function type [-Wcast-function-type-strict]
    24 | return decltype(fun_ptr)( f_ptr /*comment*/);
       |        ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``-Wzero-as-null-pointer-constant`` diagnostic is no longer emitted when using ``__null``
  (or, more commonly, ``NULL`` when the platform defines it as ``__null``) to be more consistent
  with GCC.
- Clang will warn on deprecated specializations used in system headers when their instantiation
  is caused by user code.
- Clang will now print ``static_assert`` failure details for arithmetic binary operators.
  Example:

  .. code-block:: cpp

    static_assert(1 << 4 == 15);

  will now print:

  .. code-block:: text

    error: static assertion failed due to requirement '1 << 4 == 15'
       48 | static_assert(1 << 4 == 15);
          |               ^~~~~~~~~~~~
    note: expression evaluates to '16 == 15'
       48 | static_assert(1 << 4 == 15);
          |               ~~~~~~~^~~~~

- Clang now diagnoses definitions of friend function specializations, e.g. ``friend void f<>(int) {}``.
- Clang now diagnoses narrowing conversions involving const references.
  (`#63151: <https://github.com/llvm/llvm-project/issues/63151>`_).


Improvements to Clang's time-trace
----------------------------------
- Two time-trace scope variables are added. A time trace scope variable of
  ``ParseDeclarationOrFunctionDefinition`` with the function's source location
  is added to record the time spent parsing the function's declaration or
  definition. Another time trace scope variable of ``ParseFunctionDefinition``
  is also added to record the name of the defined function.

Bug Fixes in This Version
-------------------------
- Fixed an issue where a class template specialization whose declaration is
  instantiated in one module and whose definition is instantiated in another
  module may end up with members associated with the wrong declaration of the
  class, which can result in miscompiles in some cases.
- Fix crash on use of a variadic overloaded operator.
  (`#42535 <https://github.com/llvm/llvm-project/issues/42535>`_)
- Fix a hang on valid C code passing a function type as an argument to
  ``typeof`` to form a function declaration.
  (`#64713 <https://github.com/llvm/llvm-project/issues/64713>`_)
- Clang now reports missing-field-initializers warning for missing designated
  initializers in C++.
  (`#56628 <https://github.com/llvm/llvm-project/issues/56628>`_)
- Clang now respects ``-fwrapv`` and ``-ftrapv`` for ``__builtin_abs`` and
  ``abs`` builtins.
  (`#45129 <https://github.com/llvm/llvm-project/issues/45129>`_,
  `#45794 <https://github.com/llvm/llvm-project/issues/45794>`_)
- Fixed an issue where accesses to the local variables of a coroutine during
  ``await_suspend`` could be misoptimized, including accesses to the awaiter
  object itself.
  (`#56301 <https://github.com/llvm/llvm-project/issues/56301>`_)
  The current solution may bring performance regressions if the awaiters have
  non-static data members. See
  `#64945 <https://github.com/llvm/llvm-project/issues/64945>`_ for details.
- Clang now prints unnamed members in diagnostic messages instead of giving an
  empty ''. Fixes
  (`#63759 <https://github.com/llvm/llvm-project/issues/63759>`_)
- Fix crash in __builtin_strncmp and related builtins when the size value
  exceeded the maximum value representable by int64_t. Fixes
  (`#64876 <https://github.com/llvm/llvm-project/issues/64876>`_)
- Fixed an assertion if a function has cleanups and fatal erors.
  (`#48974 <https://github.com/llvm/llvm-project/issues/48974>`_)
- Clang now emits an error if it is not possible to deduce array size for a
  variable with incomplete array type.
  (`#37257 <https://github.com/llvm/llvm-project/issues/37257>`_)
- Clang's ``-Wunused-private-field`` no longer warns on fields whose type is
  declared with ``[[maybe_unused]]``.
  (`#61334 <https://github.com/llvm/llvm-project/issues/61334>`_)
- For function multi-versioning using the ``target``, ``target_clones``, or
  ``target_version`` attributes, remove comdat for internal linkage functions.
  (`#65114 <https://github.com/llvm/llvm-project/issues/65114>`_)
- Clang now reports ``-Wformat`` for bool value and char specifier confusion
  in scanf. Fixes
  (`#64987 <https://github.com/llvm/llvm-project/issues/64987>`_)
- Support MSVC predefined macro expressions in constant expressions and in
  local structs.
- Correctly parse non-ascii identifiers that appear immediately after a line splicing
  (`#65156 <https://github.com/llvm/llvm-project/issues/65156>`_)
- Clang no longer considers the loss of ``__unaligned`` qualifier from objects as
  an invalid conversion during method function overload resolution.
- Fix lack of comparison of declRefExpr in ASTStructuralEquivalence
  (`#66047 <https://github.com/llvm/llvm-project/issues/66047>`_)
- Fix parser crash when dealing with ill-formed objective C++ header code. Fixes
  (`#64836 <https://github.com/llvm/llvm-project/issues/64836>`_)
- Clang now allows an ``_Atomic`` qualified integer in a switch statement. Fixes
  (`#65557 <https://github.com/llvm/llvm-project/issues/65557>`_)

- Clang will no longer diagnose an erroneous non-dependent ``switch`` condition
  during instantiation, and instead will only diagnose it once, during checking
  of the function template.

Bug Fixes to Compiler Builtins
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Bug Fixes to Attribute Support
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Bug Fixes to C++ Support
^^^^^^^^^^^^^^^^^^^^^^^^

- Fix crash when using lifetimebound attribute in function with trailing return.
  Fixes (`#73619 <https://github.com/llvm/llvm-project/issues/73619>`_)
- Addressed an issue where constraints involving injected class types are perceived
  distinct from its specialization types.
  (`#56482 <https://github.com/llvm/llvm-project/issues/56482>`_)
- Fixed a bug where variables referenced by requires-clauses inside
  nested generic lambdas were not properly injected into the constraint scope.
  (`#73418 <https://github.com/llvm/llvm-project/issues/73418>`_)
- Fixed a crash where substituting into a requires-expression that refers to function
  parameters during the equivalence determination of two constraint expressions.
  (`#74447 <https://github.com/llvm/llvm-project/issues/74447>`_)
- Fixed deducing auto& from const int in template parameters of partial
  specializations.
  (`#77189 <https://github.com/llvm/llvm-project/issues/77189>`_)
- Fix for crash when using a erroneous type in a return statement.
  Fixes (`#63244 <https://github.com/llvm/llvm-project/issues/63244>`_)
  and (`#79745 <https://github.com/llvm/llvm-project/issues/79745>`_)
- Fix incorrect code generation caused by the object argument of ``static operator()`` and ``static operator[]`` calls not being evaluated.
  Fixes (`#67976 <https://github.com/llvm/llvm-project/issues/67976>`_)
- Fix crash and diagnostic with const qualified member operator new.
  Fixes (`#79748 <https://github.com/llvm/llvm-project/issues/79748>`_)
- Fixed a crash where substituting into a requires-expression that involves parameter packs
  during the equivalence determination of two constraint expressions.
  (`#72557 <https://github.com/llvm/llvm-project/issues/72557>`_)
- Fix a crash when specializing an out-of-line member function with a default
  parameter where we did an incorrect specialization of the initialization of
  the default parameter.
  Fixes (`#68490 <https://github.com/llvm/llvm-project/issues/68490>`_)
- Fix a crash when trying to call a varargs function that also has an explicit object parameter.
  Fixes (`#80971 ICE when explicit object parameter be a function parameter pack`)
- Reject explicit object parameters on `new` and `delete` operators.
  Fixes (`#82249 <https://github.com/llvm/llvm-project/issues/82249>` _)
- Fixed a bug where abbreviated function templates would append their invented template parameters to
  an empty template parameter lists.
- Clang now classifies aggregate initialization in C++17 and newer as constant
  or non-constant more accurately. Previously, only a subset of the initializer
  elements were considered, misclassifying some initializers as constant. Fixes
  some of (`#80510 <https://github.com/llvm/llvm-project/issues/80510>`).
- Clang now ignores top-level cv-qualifiers on function parameters in template partial orderings.
  (`#75404 <https://github.com/llvm/llvm-project/issues/75404>`_)
- No longer reject valid use of the ``_Alignas`` specifier when declaring a
  local variable, which is supported as a C11 extension in C++. Previously, it
  was only accepted at namespace scope but not at local function scope.
- Clang no longer tries to call consteval constructors at runtime when they appear in a member initializer.
  (`#82154 <https://github.com/llvm/llvm-project/issues/82154>`_`)
- Fix crash when using an immediate-escalated function at global scope.
  (`#82258 <https://github.com/llvm/llvm-project/issues/82258>`_)
- Correctly immediate-escalate lambda conversion functions.
  (`#82258 <https://github.com/llvm/llvm-project/issues/82258>`_)
- Fixed an issue where template parameters of a nested abbreviated generic lambda within
  a requires-clause lie at the same depth as those of the surrounding lambda. This,
  in turn, results in the wrong template argument substitution during constraint checking.
  (`#78524 <https://github.com/llvm/llvm-project/issues/78524>`_)
- Clang no longer instantiates the exception specification of discarded candidate function
  templates when determining the primary template of an explicit specialization.
- Fixed a crash in Microsoft compatibility mode where unqualified dependent base class
  lookup searches the bases of an incomplete class.
- Fix a crash when an unresolved overload set is encountered on the RHS of a ``.*`` operator.
  (`#53815 <https://github.com/llvm/llvm-project/issues/53815>`_)
- In ``__restrict``-qualified member functions, attach ``__restrict`` to the pointer type of
  ``this`` rather than the pointee type.
  Fixes (`#82941 <https://github.com/llvm/llvm-project/issues/82941>`_),
  (`#42411 <https://github.com/llvm/llvm-project/issues/42411>`_), and
  (`#18121 <https://github.com/llvm/llvm-project/issues/18121>`_).
- Clang now properly reports supported C++11 attributes when using
  ``__has_cpp_attribute`` and parses attributes with arguments in C++03
  (`#82995 <https://github.com/llvm/llvm-project/issues/82995>`_)
- Clang now properly diagnoses missing 'default' template arguments on a variety
  of templates. Previously we were diagnosing on any non-function template
  instead of only on class, alias, and variable templates, as last updated by
  CWG2032. Fixes (#GH83461)
- Fixed an issue where an attribute on a declarator would cause the attribute to
  be destructed prematurely. This fixes a pair of Chromium that were brought to
  our attention by an attempt to fix in (#GH77703). Fixes (#GH83611).

Bug Fixes to AST Handling
^^^^^^^^^^^^^^^^^^^^^^^^^
- Fixed an import failure of recursive friend class template.
  `Issue 64169 <https://github.com/llvm/llvm-project/issues/64169>`_
- Remove unnecessary RecordLayout computation when importing UnaryOperator. The
  computed RecordLayout is incorrect if fields are not completely imported and
  should not be cached.
  `Issue 64170 <https://github.com/llvm/llvm-project/issues/64170>`_

Miscellaneous Bug Fixes
^^^^^^^^^^^^^^^^^^^^^^^

Miscellaneous Clang Crashes Fixed
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
- Fixed a crash when parsing top-level ObjC blocks that aren't properly
  terminated. Clang should now also recover better when an @end is missing
  between blocks.
  `Issue 64065 <https://github.com/llvm/llvm-project/issues/64065>`_
- Fixed a crash when check array access on zero-length element.
  `Issue 64564 <https://github.com/llvm/llvm-project/issues/64564>`_

OpenACC Specific Changes
------------------------
- OpenACC Implementation effort is beginning with semantic analysis and parsing
  of OpenACC pragmas. The ``-fopenacc`` flag was added to enable these new,
  albeit incomplete changes. The ``_OPENACC`` macro is currently defined to
  ``1``, as support is too incomplete to update to a standards-required value.
- Added ``-fexperimental-openacc-macro-override``, a command line option to
  permit overriding the ``_OPENACC`` macro to be any digit-only value specified
  by the user, which permits testing the compiler against existing OpenACC
  workloads in order to evaluate implementation progress.

OpenACC Specific Changes
------------------------
- OpenACC Implementation effort is beginning with semantic analysis and parsing
  of OpenACC pragmas. The ``-fopenacc`` flag was added to enable these new,
  albeit incomplete changes. The ``_OPENACC`` macro is currently defined to
  ``1``, as support is too incomplete to update to a standards-required value.
- Added ``-fexperimental-openacc-macro-override``, a command line option to
  permit overriding the ``_OPENACC`` macro to be any digit-only value specified
  by the user, which permits testing the compiler against existing OpenACC
  workloads in order to evaluate implementation progress.

Target Specific Changes
-----------------------

AMDGPU Support
^^^^^^^^^^^^^^
- Use pass-by-reference (byref) in stead of pass-by-value (byval) for struct
  arguments in C ABI. Callee is responsible for allocating stack memory and
  copying the value of the struct if modified. Note that AMDGPU backend still
  supports byval for struct arguments.

X86 Support
^^^^^^^^^^^

- Added option ``-m[no-]evex512`` to disable ZMM and 64-bit mask instructions
  for AVX512 features.

Arm and AArch64 Support
^^^^^^^^^^^^^^^^^^^^^^^

Android Support
^^^^^^^^^^^^^^^

- Android target triples are usually suffixed with a version. Clang searches for
  target-specific runtime and standard libraries in directories named after the
  target (e.g. if you're building with ``--target=aarch64-none-linux-android21``,
  Clang will look for ``lib/aarch64-none-linux-android21`` under its resource
  directory to find runtime libraries). If an exact match isn't found, Clang
  would previously fall back to a directory without any version (which would be
  ``lib/aarch64-none-linux-android`` in our example). Clang will now look for
  directories for lower versions and use the newest version it finds instead,
  e.g. if you have ``lib/aarch64-none-linux-android21`` and
  ``lib/aarch64-none-linux-android29``, ``-target aarch64-none-linux-android23``
  will use the former and ``-target aarch64-none-linux-android30`` will use the
  latter. Falling back to a versionless directory will now emit a warning, and
  the fallback will be removed in Clang 19.

Windows Support
^^^^^^^^^^^^^^^
- Fixed an assertion failure that occurred due to a failure to propagate
  ``MSInheritanceAttr`` attributes to class template instantiations created
  for explicit template instantiation declarations.

- The ``-fno-auto-import`` option was added for MinGW targets. The option both
  affects code generation (inhibiting generating indirection via ``.refptr``
  stubs for potentially auto imported symbols, generating smaller and more
  efficient code) and linking (making the linker error out on such cases).
  If the option only is used during code generation but not when linking,
  linking may succeed but the resulting executables may expose issues at
  runtime.

LoongArch Support
^^^^^^^^^^^^^^^^^

RISC-V Support
^^^^^^^^^^^^^^
- Unaligned memory accesses can be toggled by ``-m[no-]unaligned-access`` or the
  aliases ``-m[no-]strict-align``.

CUDA/HIP Language Changes
^^^^^^^^^^^^^^^^^^^^^^^^^

CUDA Support
^^^^^^^^^^^^

AIX Support
^^^^^^^^^^^

- Introduced the ``-maix-small-local-exec-tls`` option to produce a faster
  access sequence for local-exec TLS variables where the offset from the TLS
  base is encoded as an immediate operand.
  This access sequence is not used for TLS variables larger than 32KB, and is
  currently only supported on 64-bit mode.

WebAssembly Support
^^^^^^^^^^^^^^^^^^^

AVR Support
^^^^^^^^^^^

DWARF Support in Clang
----------------------

Floating Point Support in Clang
-------------------------------
- Add ``__builtin_elementwise_log`` builtin for floating point types only.
- Add ``__builtin_elementwise_log10`` builtin for floating point types only.
- Add ``__builtin_elementwise_log2`` builtin for floating point types only.
- Add ``__builtin_elementwise_exp`` builtin for floating point types only.
- Add ``__builtin_elementwise_exp2`` builtin for floating point types only.
- Add ``__builtin_set_flt_rounds`` builtin for X86, x86_64, Arm and AArch64 only.
- Add ``__builtin_elementwise_pow`` builtin for floating point types only.
- Add ``__builtin_elementwise_bitreverse`` builtin for integer types only.
- Add ``__builtin_elementwise_sqrt`` builtin for floating point types only.
- ``__builtin_isfpclass`` builtin now supports vector types.
- ``#pragma float_control(precise,on)`` enables precise floating-point
  semantics. If ``math-errno`` is disabled in the current TU, clang will
  re-enable ``math-errno`` in the presense of
  ``#pragma float_control(precise,on)``.
- Add ``__builtin_exp10``, ``__builtin_exp10f``,
  ``__builtin_exp10f16``, ``__builtin_exp10l`` and
  ``__builtin_exp10f128`` builtins.

AST Matchers
------------
- Add ``convertVectorExpr``.
- Add ``dependentSizedExtVectorType``.
- Add ``macroQualifiedType``.

clang-format
------------
- Add ``AllowBreakBeforeNoexceptSpecifier`` option.

libclang
--------

- Exposed arguments of ``clang::annotate``.

Static Analyzer
---------------

- Added a new checker ``core.BitwiseShift`` which reports situations where
  bitwise shift operators produce undefined behavior (because some operand is
  negative or too large).

- Fix false positive in mutation check when using pointer to member function.
  (`#66204: <https://github.com/llvm/llvm-project/issues/66204>`_).

- The ``alpha.security.taint.TaintPropagation`` checker no longer propagates
  taint on ``strlen`` and ``strnlen`` calls, unless these are marked
  explicitly propagators in the user-provided taint configuration file.
  This removal empirically reduces the number of false positive reports.
  Read the PR for the details.
  (`#66086 <https://github.com/llvm/llvm-project/pull/66086>`_)

.. _release-notes-sanitizers:

Sanitizers
----------

- ``-fsanitize=signed-integer-overflow`` now instruments ``__builtin_abs`` and
  ``abs`` builtins.

Python Binding Changes
----------------------

Additional Information
======================

A wide variety of additional information is available on the `Clang web
page <https://clang.llvm.org/>`_. The web page contains versions of the
API documentation which are up-to-date with the Git version of
the source code. You can access versions of these documents specific to
this release by going into the "``clang/docs/``" directory in the Clang
tree.

If you have any questions or comments about Clang, please feel free to
contact us on the `Discourse forums (Clang Frontend category)
<https://discourse.llvm.org/c/clang/6>`_.
