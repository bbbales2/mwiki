# Running Tests from the math library

Running tests in the Stan Math Library require these libraries:

- Boost
- Eigen
- Google Test
- CppLint (optional)

If the Stan Math Library has been cloned as part of the Stan library, no configuration is necessary.

If the Stan Math Library is outside of the Stan library, the following variables have to be provided to make in either `make/local` or `~/.config/stan/make.local`:

- `STANAPI_HOME` (the home directory of the Stan library, trailing slash is necessary!)

or

- `EIGEN`
- `BOOST`
- `GTEST`
- `CPPLINT` (optional)

Example `~/.config/stan/make.local` file:
```
STANAPI_HOME = ~/stan/
```
**NOTE** must include the trailing slash

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
