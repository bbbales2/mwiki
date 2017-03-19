If you have a function f and you know the gradients for it, it is straightforward to add the function to Stan's math library.


#### Templated function

If your function is templated separately on all of its arguments and bottoms out only in functions built into the C++ standard library or into Stan, there's nothing to do other than add it to the Stan namespace.  For example, if we want to code in a simple z-score calculation, we can do it very easily as follows, because the `mean` and `sd` function are built into Stan's math library:


```
#include <stan/math/rev.hpp>

namespace stan {
  namespace math {
    vector<T> z(const vector<T>& x) {
      T mean = mean(x);
      T sd = sd(x);
      vector<T> result;
      for (size_t i = 0; i < x.size(); ++i)
        result[i] = (x[i] - mean) / sd;
      return result;
    }
  }
}
```

#### Simple univariate example with known derivatives

Suppose have a code to calculate a univariate function and its derivative:

```
namespace bar {
  double foo(double x);
  double d_foo(double x);  
}
```

We can implement foo for Stan as follows, making sure to put it in the `stan::math` namespace so that argument-dependnet lookup (ADL) can work.  We'll assume the two relevant functions are defined in `my_foo.hpp`

```
#include "my_foo.hpp"
#include <stan/math/rev.hpp>

namespace stan {
  namespace math {
    double foo(double x) {
      return bar::foo(x);
    }

    var foo(const var& x) {
      double a = x.val();
      double fa = bar::foo(x_d);
      double dfa_da = bar::d_foo(a);
      return precomp_v_vari(fa, x.vi_, dfa_da);
    }
  }
}
```

There are similar functions `precomp_vv_vari` for two-argument functions and so on.



#### Functions of more than one argument

For most efficiency, each argument should independently allow `var` or `double` arguments (`int` arguments don't need gradients---those can just get carried along).  So if you're writing a two-argument function of scalars, you want to do this:

```
namespace bar {
  double foo(double x, double y);
  double dfoo_dx(double x, double y);
  double dfoo_dy(double x, double y);
}
```

This can be coded as follows

```
#include "my_foo.hpp"
#include <stan/math/rev.hpp>

namespace stan {
  namespace math {
    double foo(double x, double y) {
      return bar::foo(x, y);
    }

    var foo(const var& x, double y) {
      double a = x.val();
      double fay = bar::foo(a, y);
      double dfay_da = bar::foo_dx(a, y);
      return return precomp_v_vari(fay, x.vi_, dfay_da);
    }

    var foo(double x, const var& y) {
      double b = y.val();
      double fxb = bar::foo(x, b);
      double dfxb_db = bar::foo_dy(x, b);
      return return precomp_v_vari(fxb, y.vi_, dfxb_db);
    }
   
    var foo(const var& x, const var& y) {
      double a = x.val();
      double b = y.val();
      double fab = bar::foo(a, b);
      double dfab_da = bar::foo_dx(a, b);
      double dfab_db = bar::foo_dy(a, b);
      return precomp_vv_vari(fab, x.vi_, y.vi_, dfab_da, dfab_db);
    }
  }
}
```

Same thing works for three scalars.


#### Vector function with one output

Now suppose the signatures you have give you a function and a gradient using standard vectors (you can also use Eigen vectors or matrices in a similar way):

```
namespace bar {
  double foo(const vector<double>& x);
  vector<double> grad_foo(const vector<double>& x);
}
```

Of course, it's often more efficient to compute these at the same time---if you can do that, replace the lines for `fa` and `grad_fa` witha more efficient call.  Otherwise, the code follows the previous version:

```
#include "my_foo.hpp"
#include <stan/math/rev.hpp>

namespace stan {
  namespace math {
    vector<double> foo(const vector<double>& x) {
      return bar::foo(x);
    }

    vector<var> foo(const vector<var>& x) {
      vector<double> a = value_of(a);
      double fa = bar::foo(a);
      vector<double> grad_fa = bar::grad_foo(a);  
      return precomputed_gradients(fa, x, grad_fa);
    }
  }
}
```

The `precomputed_gradients` class can be used to deal with any form of inputs and outputs.  All you need to do is pack all the `var` arguments into one vector and their matching gradients into another.

#### Functions with more than one output

If you have a function with more than one output, you'll need the full Jacobian matrix.  You create each output `var` the same way you create a univariate output, using `precomputed_gradients`.  Then you just put them all together.  This time, I'll use Eigen matrices assuming that's how you get your Jacobian:

```
namespace bar {
  Eigen::VectorXd foo(const Eigen::VectorXd& x);
  Eigen::MatrixXd foo_jacobian(const Eigen::VectorXd& x);
}
```

You can code this up the same way as before (with the same includes):

```
namespace stan {
  namespace math {
    Eigen::VectorXd foo(const Eigen::VectorXd& x) {
      return bar::foo(x);
    }
    Eigen::Matrix<var, -1, 1> foo(const Eigen::Vector<var, -1, 1>& x) {
      Eigen::VectorXd a = value_of(x);
      Eigen::MatrixXd J = foo_jacobian(a);
      Eigen::Matrix<var, -1, 1> result(a.rows());
      std::vector<double> grads(x.rows());
      for (int i = 0; i < a.rows(); ++i) {
        for (int j = 0; j < a.rows(); ++j)
          grads(j) = J(i, j);
        result(i) = precomputed_gradients(a(i), x, grads);
      }
      return result;
    }
  }
}
```

We cold obviously use a form of `precomputed_gradients` that takes in an entire Jacobian to remove the error-prone and non-memory-local loops (by default, matrices are stored column-major in Eigen).
   

  