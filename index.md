# Making Smart Pointer Checkers default checkers in the Static Analyzer

## Overview 
This project aims to complete the `SmartPtrChecker` and thus `SmartPtrModeling` checkers to detect null-dereferences of the `std::unique_ptr`. This is a continuation of the GSoC 2020 [project](https://docs.google.com/document/d/1WZSt45kZUhg0UbOv0HXBhyEYaHrb-G-TpEhj_nU041Q/edit) in the same area.

## Description
`SmartPtrChecker` used to be a checker in the Clang Static Analyzer for detecting "simple" null dereferences. The GSoC 2020 project improved upon this to provide the `SmartPtrModeling` checker, which models many of the important operations on `std::unique_ptr`, such as move, reset, release, convert to bool, etc. This project completes most of the gaps and uncompleted areas in the modelling, including the ubiquitous `std::make_unique` function and the destructor (which communicates with the `MallocChecker` to remove false positives). *Currently, we are a few bug fixes and some polishing away from making this a default checker* (as of Clang 13, it is an alpha checker).

## Revisions

One aim of this project was to make sure that no method defined on `std::unique_ptr` is left un-modelled. If we don't, CSA either does:

- *Conservative evaluation*, which leads to loss of information and possibly warnings being suppressed
- *Inlining*, which sometimes leads the CSA to falsely believe that there is a bug in the standard library code (because we have modelled parts of it and are inlining other parts). Since bugs in standard library code are normally suppressed, this leads to other bugs beings suppresed (false-negatives). For example:

```cpp
void foo() {
    auto smart = std::unique_ptr<int>(new int(13));
    auto raw = new int(29);
    // There is a leak here.
}
```
Without [D105821](https://reviews.llvm.org/D105821), this used to have **no** leak warning. The inlining of the destructor of `std::unique_ptr` led the CSA to falsely believe there was a bug in the destructor code.

### Model comparision methods of `std::unique_ptr`

[D104616](https://reviews.llvm.org/D104616) introduces modelling of comparision operator overloads of `std::unique_ptr`. It leverages the `SValBuilder::evalBinOp` to evaluate the result of the operator. More importantly, it attempts to perform a **state-split** if it is possible.

### Model `std::swap` specialization for `std::unique_ptr`

[D104300](https://reviews.llvm.org/D104300) models the `std::swap` specialization for `std::unique_ptr`. There is an existing `swap` method on `std::unique_ptr`, which performs roughly the same thing. Thus, the common code is refactored out and both methods are handled exactly the same way.

![std-swap](assets/std-swap.png)


### Model `<<` operator specialization for `std::unique_ptr`

[D105421](https://reviews.llvm.org/D105421) models the `<<` operator specialization for `std::unique_ptr`. There is not that much modelling required for this method, other than invalidating the stream region.


### Model `std::make_unique` and cousins

[D103750](https://reviews.llvm.org/D103750) models the quintessential `std::make_unique` function. In this entire checker, we only account for `unique_ptr` containing pointers, not arrays - we do the same here as well. The crux of this patch is informing the CSA that we are constructing an object via this function. Ideally this should be handled automatically (as it is for constructors). But here we bail out and simply call `ExprEngine::updateObjectsUnderConstruction`. Also we ensure that the `ProgramState` knows that we have a non-null inner pointer in the `unique_ptr`.


### Fix for faulty namespace test and add flag checking

[D106296](https://reviews.llvm.org/D106296) prevents a crash we encountered by handling the case where we do not have the declaration of a function and so cannot know its namespace. Also we add the `ModelSmartPtrDereference` flag check to the previously modelled functions.


### Add option to `SATest.py` for extra checkers

[D106739](https://reviews.llvm.org/D106739) augments the functionality of the `SATest.py` script. This script runs the CSA on the projects in the `clang/utils/analyzer/projects` directory (via a Docker image). By default, the script runs the CSA with only default checkers enabled. This patch adds a flag to enable extra checkers (for our case, the `SmartPtrChecker`), enabling us to conveniently run test our patches.


### Model destructor for `std::unique_ptr`

[D105821](https://reviews.llvm.org/D105821) models the elephant in the room - the destructor for `std::unique_ptr`. Although we are just modelling the destructor in this patch, in reality, we have ended up calling the `MallocChecker` and improving the invalidation scheme in addition to the modelling. We are doing the following steps:

- Enable calling `evalCall` for destructor-calls (before this patch, only the `defaultEvalCall` is run)
- Escape inner pointer on construction and reset
- Escape all reachable symbols in `checkRegionChange`
- *Invalidate* the inner pointer and remove it from `TrackedRegionMap` (in both destructor and in reset)

The fuss about *invalidation* of inner pointer stems from the fact that `delete` also calls the *member destructor* (if it exists). Consider the following class:

```cpp
class LameVector {
    size_t sz = 0;
    size_t cap = 4;
    size_t el_sz = 1;
    unsigned char *buf;

public:
    LameVector() : buf{new unsigned char[cap]} {}
    ~LameVector() { delete[] buf; }
};

void foo() {
    auto ptr = std::make_unique<LameVector>();
}
```

In this code, `LameVector` contains an inner pointer, which is freed in its destructor. If we do not invalidate this inner pointer, CSA will think `LameVector` and hence `ptr` leaks memory.

This is a stop gap measure and it seems to work for the time being. The proper solution is to inline both the member constructor (both during constructor calls and during `make_unique`) and the member destructor. But we don't yet have a mechanism to "evaluate" functions from a checker.