# Using the Stan Math Library

If you're using the Stan Math Library, you should only include one header file. The most common one to include is `stan/math.hpp`.

To choose the correct include, you'll need to know two things:
- primitive (no auto-diff) vs reverse-mode autodiff vs forward-mode autodiff vs mixed autodiff (includes both reverse and forward-mode)
- scalar (no `std::vector` or `Eigen::Matrix`) vs array (`std::vector`, but no `Eigen::Matrix`) vs matrix (includes array)

Once you picked what you want to use, you'll include:
- `stan/math/prim/scal.hpp` for primitive and scalar
- `stan/math/prim/arr.hpp` for primitive and array
- `stan/math/prim/mat.hpp` for primitive and matrix
- `stan/math/rev/scal.hpp` for reverse-mode and scalar
- `stan/math/rev/arr.hpp` for reverse-mode and array
- `stan/math/rev/mat.hpp` for reverse-mode and matrix. This is what's included when you include `stan/math.hpp`
- `stan/math/fwd/scal.hpp` for forward-mode and scalar
- `stan/math/fwd/arr.hpp` for forward-mode and array
- `stan/math/fwd/mat.hpp` for forward-mode and matrix
- `stan/math/mix/scal.hpp` for mixed-mode and scalar
- `stan/math/mix/arr.hpp` for mixed-mode and array
- `stan/math/mix/mat.hpp` for mixed-mode and matrix.

## Order Dependencies for Eigen Traits

The reason we provide these files is because we need to make sure that we instantiate traits in the correct order for the Eigen traits and the `std::numeric_limits` traits. If we try to specialize a template after instantiating the template, then it's too late and it won't compile or we'll have other issues.  See this discussion for more details:

http://stackoverflow.com/questions/12732042/explicit-specialization-of-stditerator-traitschar-after-instantiation-c


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

To run the multiple translation unit tests, type:
```
> ./runTests.py test/unit/multiple_translation_units_test.cpp
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

To test headers,
```
> make test-headers
```

