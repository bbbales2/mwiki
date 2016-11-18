Editing this page.

# Distribution Tests

We've written a framework for testing our univariate distribution functions. It's not well doc'ed and the framework is pretty complicated, but it is used to make sure our distributions are ok. (Well -- ok enough.)

This is an attempt to document how to use the framework and how we ended up with this framework.

## How

In this section, I'll describe how to run the tests and what's happening.

### How to Run All the Distribution Tests (for the impatient)

From within the Math library:

```
./runTests.py test/prob

```

This will take hours, perhaps many hours. Most of the time is spent compiling. You might want to add the parallel flag to the python script.


### Steps Involved in Running All the Tests

Here, I'm just going to describe the steps that are taken to run the tests. Details on how to write a test and what's inside the framework are further down. These are the steps taken when calling `./runTests.py test/prob` -- just broken out step by step.

1. `make generate-tests` is called.
    1. This first builds the `test/prob/generate_tests` executable from `test/prob/generate_tests.cpp`.
    2. For each test file inside `test/prob/*/*`, it will call the executable with the test file as the first argument and the number of template instantiations per file within the second argument. For example, for testing the `bernoulli_log()` function, make will call: `test/prob/generate_tests test/prob/bernoulli/bernoulli_test.hpp 100`
    The call to the executable will generate 5 different test files, all within the `test/prob/bernoulli/` folder:
        - bernoulli\_00000\_generated\_fd\_test.cpp
        - bernoulli\_00000\_generated\_ffd\_test.cpp
        - bernoulli\_00000\_generated\_ffv\_test.cpp
        - bernoulli\_00000\_generated\_fv\_test.cpp
        - bernoulli\_00000\_generated\_v\_test.cpp
    These generated files are actual unit tests and can be run just like any other test in the math library. The `{fd, ffd, ffv, fv, v}` are the types of instantiations within the file. Those types map to: `fvar<double>`, `fvar<fvar<double>>`, `fvar<fvar<var>>`, `fvar<var>`, `var`.
    In disributions with many arguments, there will be more than 1 file per types of instantiations. 
2. Each of the test files within `test/prob/*/*` are compiled.
3. Each of the test files within `test/prob/*/*` is run.


### How to Run a Single Distribution Test?
