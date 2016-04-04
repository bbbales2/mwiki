# Where do I create a new issue

Stan's development is across multiple repositories. See this page for details on where to put new issues:
https://github.com/stan-dev/stan/wiki/Where-do-I-create-a-new-issue

# Running tests from the math library

Running tests in the Stan Math Library require these libraries, which are all included in the repository:

- Boost
- Eigen
- Google Test
- CppLint (optional)

No additional configuration is necessary to start running with the default libraries. 

If you want to use custom locations for the library locations, set these makefile variables:

- `EIGEN`
- `BOOST`
- `GTEST`
- `CPPLINT` (optional)

Example `~/.config/stan/make.local` file:
```
BOOST = ~/boost
```

# Running tests

To run tests, you will need a copy of the Math library, a C++ compiler, make, and python (for the test script).

To run the unit tests, type:
```
> ./runTests.py test/unit
```

To run the auto-generated distribution tests, type:
```
> ./runTests.py test/prob
```


If you see this message:

```
------------------------------------------------------------
make generate-tests -s
test/prob/generate_tests.cpp:9:10: fatal error: 'boost/algorithm/string.hpp' file not found
#include <boost/algorithm/string.hpp>
         ^
1 error generated.
make: *** [test/prob/generate_tests] Error 1
make generate-tests -s failed
```

the library paths have not been configured correctly.

# Order Dependencies for Eigen Traits

We need to make sure that for the Eigen traits and the `std::numeric_limits` traits, that we don't try to specialize a template after instantiating the template.  By then it's too late.  See:

http://stackoverflow.com/questions/12732042/explicit-specialization-of-stditerator-traitschar-after-instantiation-c

In practice, this means for stan math that 

* if you are using reverse-mode autodiff, you need to include `stan/math/rev/mat/fun/Eigen_NumTraits.hpp` before including anything from the Eigen namespace (e.g., can't include anything from `prim/mat/*` before `rev/*` if you are going to include `rev/mat/*`), and

* if you are using forward-mode autodiff, you need to include `stan/math/rev/mat/fwd/Eigen_NumTraits.hpp`.





