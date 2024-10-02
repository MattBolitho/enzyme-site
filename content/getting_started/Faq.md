---
title: "FAQ"
date: "2019-11-29"
menu: "main"
weight: 40
---

## Enzyme builds successfully but won't run tests

Double check that Enzyme's build can find lit, LLVM's testing framework. This can be done by explicitly passing lit as an argument to CMake as described [here](/Installation).

## Enzyme is Crashing

LLVM's plugin infrastructure is broken in many versions. Empirically LLVM 8 and up will often incorrectly disallow memory from being passed between LLVM and a plugin. If you see one of these errors and want to use the same version of LLVM try passing the flag `enzyme_preopt=0` described [here](/getting_started/UsingEnzyme). The flag disables preprocessing optimizations that Enzyme runs and tends to reduce these errors. If this doesn't work, check the following.

## I receive an `__enzime_autodiff` undefined symbol either at runtime or at compile time
This error means that you have Enzyme calls which did not run the the Enzyme AD transformation. Please check that you have Enzyme run as part of the compilation which resulted in the error.
In projects with multisource and/or complex building systems it's possible to overlook one source file being compiled/linked without the appropriate Enzyme plugin pass.
When using LLDEnzyme and LTO, make sure that **all** object file rules include the `-flto` argument and that the final linking step includes **both** `-fuse-ld=lld` and `-flto`.
Obviously, such final linking step must also include the Enzyme plugin with something like `-Wl,--load-pass-plugin=/path/to/LLDEnzyme-<VERSION>.so`

## Many errors occur, starting with `Failed to load passes from '/path/to/LLDEnzyme-<VERSION>.so'. Request ignored.`
This means that the Enzyme plugin has not been loaded. There may be various reasons for this to happen, some common ones are
* typo in the `/path/to/` part
* plugin (e.g. using `LLDEnzyme` plugin for `Clang` or `LLVM` or some other incorrect combination)
* shared object not present because its compilation failed or had been deleted
* the `/path/to` contains shells expansions not honored by the build system, for example `Makefile` may fail to expand `~` but does correctly expland `$(HOME)`

## Other

If you have an issue not resolved here, please make an issue on [Github](https://github.com/EnzymeAD/Enzyme) and consider making a pull request to this FAQ!
