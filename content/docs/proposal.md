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

### TODO’s and FIXME’s in `SmartPtrModelling`

The following table is a list of such flags (as of now) in `SmartPtrModelling.cpp`:

| Sl. No. | Line number |  Type |                                                                         Comment                                                                        |
|:-------:|:-----------:|:-----:|:-------------------------------------------------------------------------------------------------------------------------------------------------------|
|    1    |     170     |  TODO | Update CallDescription to support anonymous calls?                                                                                                     |
|    2    |     171     |  TODO | Handle other methods, such as .get() or .release(). But once we do, we'd need a visitor to explain null dereferences that are found via such modeling. |
|    3    |     191     | FIXME | Once we model std::move for smart pointers clean up this and use that modeling.                                                                        |
|    4    |     197     |  TODO | Model this case as well. At least, avoid invalidation of globals.                                                                                      |
|    5    |     202     |  TODO | Add a note to bug re\| ports describing this decision.                                                                                                 |
|    6    |     359     |  TODO | Make sure to invalidate the region in the Store if we don't have time to model all methods.                                                            |
|    7    |     394     |  TODO | Add support to enable MallocChecker to start tracking the raw pointer.                                                                                 |
|    8    |     460     |  TODO | Add NoteTag, for how the raw pointer got using 'get' method.                                                                                           |

I am currently working on item **7** ([D98726](https://reviews.llvm.org/D98726)) and item **8** ([D97183](https://reviews.llvm.org/D97183)) of the above table. Not all of these TODO’s are problematic and some of them are enhancements rather than potential corners. I would prioritize removing the most important ones first.

### Investigate into false positives

There are no documented false positives according to this [report](https://docs.google.com/document/d/1WZSt45kZUhg0UbOv0HXBhyEYaHrb-G-TpEhj_nU041Q/edit#heading=h.5mykosj4yack). That said, there is an issue with “inlined defensive checking”. So, to be certain, I intend to run Static Analyzer on some more open-source projects. It has already been tested on LLVM code and WebKit. These can be run again, as well as on Chromium and Firefox codebases, or maybe some more others. There should be no false positives and in fact some doubtful bugs are suppressed to avoid creating false positives.

*At this point, we should be able to move the checker of `std::unique_ptr` to default from alpha.*

### About `shared_ptr` and `weak_ptr`

The following table compares the methods of `std::unique_ptr` and `std::shared_ptr` (✓ indicates done, ⨯ indicates not applicable/modellable, blank indicates TODO):

|         Method         | `std::unique_ptr` | `std::shared_ptr` |                                      Remarks                                      |
|:-----------------------|:---------------:|:---------------:|:----------------------------------------------------------------------------------|
| Construct from pointer |        ✓        |                 | Similar for both                                                                  |
| Copy constructor       |        ⨯        |                 | Increases ref-count                                                               |
| Move constructor       |        ✓        |                 | Similar for both                                                                  |
| Destructor             |        ✓        |                 | Decreases ref-count                                                               |
| Copy operator          |        ⨯        |                 | Increases ref-count                                                               |
| Move operator          |        ✓        |                 | Similar for both                                                                  |
| Release                |        ✓        |        ⨯        |                                                                                   |
| Reset                  |        ✓        |                 | Resets the current `shared_ptr`, reduces ref-count of the original one (if present) |
| Swap                   |        ✓        |                 | Similar for both                                                                  |
| Get                    |        ✓        |                 | Similar for both                                                                  |
| operator bool          |        ✓        |                 | Similar for both                                                                  |

Associated with a `shared_ptr` are three “things”:

- The shared pointer object
- The inner raw pointer
- The ref-count

For determining whether a `shared_ptr` is null or not, we do not need the ref-count.

However, `std::weak_ptr` introduces a new problem with omitting the reference count. Consider the following code:

```cpp
struct A {
  int field;
  void foo() { std::cout << "Field: " << field << std::endl; }
  A(int f = 0) : field{f} {}
};

void mess() {
  std::weak_ptr<A> A_Weak;
  {
	std::shared_ptr<A> A_Shared = std::make_shared<A>(100);
	{
  	   std::shared_ptr<A> A_Copied = A_Shared;
  	   A_Weak = A_Copied;
	}
	A_Weak.lock()->foo(); // Should work
  }
  A_Weak.lock()->foo(); // Should crash
}
```

If we don’t remember the ref-count, then we cannot tell that a crash is going to happen in the second deref but not in the first one. This is because `weak_ptr` is non-owning and doesn’t prevent the deallocation of the underlying pointer (that is we can have a dangling `weak_ptr`).

### Solution: Track the raw pointer and ref count that

Both `shared_ptr` and `weak_ptr` have an underlying raw pointer (as I highlighted before). We can track that raw pointer in the GDM by a map. We can also track the ref-count in the GDM by mapping from the raw pointer to the ref-count. Thus. we have a ***UnderlyingPointerMap***, which maps from MemRegion (which corresponds to the this pointer) to an SVal (the symbolic value for the inner raw pointer). Also, we have a ***ReferenceCountMap***, which maps from the SVal to an int, the ref-count.

Thus, on construction of `shared_ptr`, one of the two things happens:

- When a `shared_ptr` is constructed from pointer or is move constructed (from `unique_ptr` or `shared_ptr`), or from `make_shared` - Add an entry to *UnderlyingPointerMap* and to *ReferenceCounterMap* (with key as 1). [If the raw pointer being used already exists in the map then we actually have a bug here, since we will have a double free on `shared_ptr `destruction].

- When a `shared_ptr` is copy-constructed, an entry to the *UnderlyingPointerMap* is made, with the value being the SVal that the original shared_ptr pointed to. Also increase the ref-count in *ReferenceCountMap*

On destruction of `shared_ptr`, we must remove the entry from *UnderlyingPointerMap* and either decrement the count in *ReferenceCounterMap* or remove the entry if it reaches 0.
The other methods will also be similarly modelled.