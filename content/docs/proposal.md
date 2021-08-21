---
title: "Proposal"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Proposal for "Making Smart Pointer Checkers default checkers in the Static Analyzer"

## Information

- **Name**: Deep Majumder
- **Email**: deep.majumder2019@gmail.com

## Problem Statement

The Clang Static Analyzer contains a `SmartPtrChecker` for catching null smart pointer dereference bugs. The data it uses to figure out such cases comes from the `SmartPtrModelling` checker, which models how a smart pointer behaves. This way, the Static Analyzer can find these bugs without accessing the source code for C++ standard library smart pointers.

However, as of now, `std::unique_ptr` has been modelled in the `SmartPtrModelling` (mostly). This leaves us with the following other standard smart pointers:

1) `std::shared_ptr`
2) `std::weak_ptr`

I intend to **enhance** and **complete** the `SmartPtrModelling` and `SmartPtrChecker` so that it is ready to move from being an experimental checker to a `default checker`. To this end I will:
1) Remove TODO’s and FIXME’s in `std::unique_ptr` as well as cover unimplemented methods.
1) Model `std::shared_ptr` and `std::weak_ptr` and cover as many possible methods.

As examples of new bugs which will be covered by this project, please see **Example 1** and **Example 2**.

**Example 1: `std::shared_ptr` constructed null**
```cpp
struct A {
    int field;
    void foo() { std::cout << "Field: " << field << "\n"; }
    A(int f = 0) : field{f} {}
};

void deref_bad(std::shared_ptr<A> AS) {
    AS->foo(); // Yikes! Null deref here
}

void maker() {
    std::shared_ptr<A> AS;
    deref_bad(AS);
}
```

**Example 2: `std::weak_ptr` used after `std::shared_ptr` it referred to is destructed**

```cpp
struct A {
  int field;
  void foo() { std::cout << "Field: " << field << "\n"; }
  A(int f = 0) : field{f} {}
};

void mess() {
  std::weak_ptr<A> A_Weak;
  {
	std::shared_ptr<A> A_Shared = std::make_shared<A>(100);
	A_Weak = A_Shared;
  } // A_Shared is dropped here
  auto trouble = A_Weak.lock(); // trouble is a null shared_ptr
  trouble->foo(); // Oops! Now that's trouble
}
```

## Goals

At the end of this project:

1) The checker for `std::unique_ptr` dereferences should be in the default checkers list
2) There should be an alpha checker for checking of `std::shared_ptr` and `std::weak_ptr`

Thus, the user will have a stable checker for checking null `std::unique_ptr` dereferences. I hope that this checker will be enabled by default.
