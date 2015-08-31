# Running Tests from the math library

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
