# Help implement forward mode automatic differentiation

We've been sitting on forward mode for a while now. It hasn't been moving forward and it would really help the state of Stan if we did. Riemmanian HMC is waiting for this to be implemented.

The purpose of this wiki is to show how to contribute. If you're looking for a way to contribute to Stan, this is an easy intro to the code base and is self-contained. We have a ton of functions that need tests. The goal of this document is to show you how to write tests to verify that a particular function is working. If that isn't clear by the end, please let someone know or fix the wiki so it is clearer.

# Overview

Right now, every Stan program that can be written gets translated to C++ and is compiled with reverse mode automatic differentiation.

We know for a fact that not every Stan program can be compiled with forward mode.

## Goals

The two goals we're tryign to achieve:

1. every Stan program is compilable with forward mode automatic differentiation
2. forward mode automatic differentiation is correct, to the best of our knowledge

By writing unit tests in the math library, we should be able to determine what functions satisfy these two goals.

## Non Goals

Although this would be nice, this is not what we're trying to do:

1. write the most efficient code for speed
2. write the most efficient code for memory
3. achieve 100% test coverage by any metric


# Tests

Unit tests in the math library will show us if the math function is compilable and hopefully whether the function is correct.

## Every function should be tested

That's every function in the math library.


## What needs to be tested?

I'm looking for 3 things:

1. the function computes the correct values for arguments that are typical
2. the function correctly handles error conditions (if any)
3. the function correctly handles boundary conditions (if any)

This should be done for these C++ types:

- `stan::math::fvar<double>`
- `stan::math::fvar<stan::math::fvar<stan::math::var>>`

To me, that's the minimal set of things that must be tested. That'll make sure that the function is compilable and that it does what we expect it to do. As you write tests, you may add additional tests. That's great. Having branch coverage is better.

## Tests for `fvar<double>`

1. Check that the value of the function is correct
2. Check that the gradient with respect to each of the continuous inputs is correct; will need to run the function multiple times to check each of the gradients.
3. Check that error conditions properly trigger errors; this should check every error condition separately
4. Check that the boundary conditions behave properly; this should check every boundary condition separately

## Tests for `fvar<fvar<var>>`

1. Check that the value of the function is correct
2. Check that the 3rd order derivative is correct
2. Check that the gradient with respect to each of the continuous inputs is correct; will need to run the function multiple times to check each of the gradients.
3. Check that error conditions properly trigger errors; this should check every error condition separately
4. Check that the boundary conditions behave properly; this should check every boundary condition separately




