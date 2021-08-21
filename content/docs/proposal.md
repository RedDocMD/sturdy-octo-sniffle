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

## Proposed Solution

### Rename the checkers

`SmartPtrModelling` should be renamed to something more reflective, like *UniquePtrModelling*.

*SmartPtrChecker* will continue to have the same name, but only `std::unique_ptr` will be covered by default in the default section of checkers. For `shared_ptr`, there will be *SharedPtrModellling* and for `weak_ptr` there will be *WeakPtrModellling*. There will be an inter-checker api for connecting the two. I would try to bring *SharedPtrModelling* to default as well, but *WeakPtrChecker* would definitely be in the alpha checkers group.

### Methods of `std::unique_ptr` not covered yet

From the [report](https://docs.google.com/document/d/1WZSt45kZUhg0UbOv0HXBhyEYaHrb-G-TpEhj_nU041Q/edit#heading=h.5mykosj4yack) of the last year’s GSoC project “Find null smart pointer dereferences with the Static Analyzer”, the following methods are yet to be modelled:
- `std::make_unique`
- `std::make_unique_for_overwrite`
- Destructor
- `std::swap`
- comparison operators

I intend to complete all of these. These will build on the existing methods in `SmartPtrModelling`.

## TODO’s and FIXME’s in `SmartPtrModelling`

The following table is a list of such flags (as of now) in `SmartPtrModelling.cpp`:

| Sl. No. | Line number | Type | Comment |
|:--------:|:-------------:|:------:|:---------|
| 1 | 170 | TODO | Update CallDescription to support anonymous calls |
| 2 | 171 | TODO | Handle other methods, such as .get() or .release(). But once we do, we'd need a visitor to explain null dereferences that are found via such modeling. |
| 3 | 191 | FIXME | Once we model std::move for smart pointers clean up this and use that modeling. |
| 4 | 197 | TODO | Model this case as well. At least, avoid invalidation of globals. |
| 5 | 202 | TODO | Add a note to bug reports describing this decision. |
| 6 | 359 | TODO | Make sure to invalidate the region in the Store if we don't have time to model all methods. |
| 7 | 394 | TODO | Add support to enable MallocChecker to start tracking the raw pointer. |
| 8 | 460 | TODO | Add NoteTag, for how the raw pointer got using 'get' method.

I am currently working on item **7** ([D98726](https://reviews.llvm.org/D98726)) and item **8** ([D97183](https://reviews.llvm.org/D97183)) of the above table. Not all of these TODO’s are problematic and some of them are enhancements rather than potential corners. I would prioritize removing the most important ones first. 