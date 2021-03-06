# Overview

We're trying to change the implementation of the ODE system for a few reasons. One of them will be for an advanced user to provide analytic Jacobians in C++. The goal will be for this to serve as both a functional and technical spec.

**NOTE: this isn't complete. It still has a lot of holes as to how it'll get implemented.**

Our goal is to refactor the system so we can move forward more quickly.

# Author

Daniel Lee. (Soon, we'll have other contributors, but this is starting out with me.) Misrepresentations are all mine.

Sebastian.
- Commented on Daniel
- Added conceptual DRAFT code (with a few deviations, main on described below)

# Goals

- Refactor the current ODE system; the `integrate_ode_*()` functions will have the same interface.
- Design the ODE system so it's understandable by other programmers.
- Design the ODE system so different (templated) classes have different functions.
- Keep the design simple.

# Nongoals

- We do not want to change the code generator. We will continue to use the generated code.
- We are not trying to optimize; we are trying for clarity.

# Scenarios (these all need more details)

- developer: new `integrate_ode_*` function using existing libraries
- developer: new `integrate_ode_*` function using a new ode solver with different requirements
- advanced user: provide Jacobian with respect to the states in C++
- advanced user: provide Jacobian with respect to the parameters in C++
- advanced user: provide Jacobian with respect to the states in Stan
- advanced user: provide Jacobian with respect to the parameters in Stan
- developer: wanting to experiment with parallelization

# An Overview

It's time to properly design how we implement ODEs under the hood. So far, the growth has followed a linear path where we were only supporting a single type of ODE system. Recently we added a second and we were able to make something work, but without a coherent design. This has caused confusion across developers that have worked on the system due to seemingly different goals, which don't have to be different goals.

Here's how we got here:

1. We started with a non-stiff Boost ODE integrator. A user would specify a Stan function as a representation of the ODE system that would be plugged in directly into the Boost ODE solver and we autodiffed the solution (super inefficient). That meant we were passing `stan::math::var`s into the solver and just going about our day. Our first attempt was just to generate code to match the requirements that Boost's ODE integrator required. It worked for simple problems. (This was Sebastian, Daniel, and Bob.)
2. Our next step was to go from the original system to a coupled ODE system. We essentially built 4 separate systems. We used one `coupled_ode_system` class to cleverly build out these four systems (there are shared computations between the different systems). The different instantiations depended on what was a parameter or data: parameters of the ODE, initial conditions. (the parameters of the ODE system could be `double`s or `var`s and the same with the initial conditions -- so 2^2 = 4 total different coupled ODE systems.)  When the parameters of the ODE and the initial conditions were both data, this would just be the ODE. Clever math was applied to generate a new system that was larger than the original system that could be solved using the existing ODE solver in `double`s and nested reverse-mode autodiff. We then constructed states with the appropriately constructed gradients from using the clever math. (This was Michael on the math, Bob on the nested autodiff, Daniel on the implementation.) 
3. We then went in a different direction with a new library for solving ODEs: Sundial's CVODES (CVODE came and went within the same version so users didn't see it). This introduced a whole new set of requirements for the differential equation that needed to be provided to it and this is where we outgrew our existing design. CVODES requires not the coupled ode system, but pieces of it separately to do solve for the ODE. We implemented a version of this using the existing pieces and jammed it together. It works, but it was really a use of our existing system that wasn't pretty. (This was mostly Sebastian and Michael on the implementation).

After the implementation of CVODES (which wasn't designed, but rather taking existing pieces and making it work), there was an attempt to refactor which actually tangled up the design making it more complex rather than simpler. It made it harder to do both rather than simpler for both.

The goal now is to design the ODE system so it works for both the non-stiff and stiff solvers. We want the design to be simple and flexible so we can move forward in a maintainable way. We want to be able to accomodate different use cases in the future (which are outlined above), without having to struggle to detangle different things.


# Details, details, details

(This is coming from my memory in talking with Sebastian. Sebastian, can you fix this so it's correct?)

We currently have two solvers with different requirements and how we use them are different.

## Terminology

- t: time
- y: states at time t, dimensionality is N
- t0: initial time
- y0: initial state at t=t0 (also dim N)
- theta: parameters needed to calculate the ode_rhs, dimensionality is M
- ode_rhs: The basic ODE right-hand-side which calculates dy/dt = f(t,y,theta)
- Jy: Jacobian of the ode_rhs wrt to the states y
- Jtheta: Jacobian of the ode_rhs wrt to the parameters theta

## Boost ODE non-stiff solver

This is implemented in the Stan math library as `integrate_ode_rk45()`.

The Boost ode solver accepts one system equation. The way we deal with this is to provide a coupled ode system. This will require jamming both ode_rhs, Jy, and Jtheta into one system equation.

## Sundials CVODES stiff solver

This is implemented in the Stan math library as `integrate_ode_bdf()`.

The CVODES solver requires the ode_rhs as one thing. The sensitivity RHS is needed separately such that Jy and Jtheta is needed as a second thing.

For the stiff solver of CVODES we also need Jy in any case. That is, for stiff integration Jy is needed even if no sensitivity calculations are done.

## Design

Instead of jamming all these things together, we're going to do this:

- ode_rhs is provided as a functor by the Stan language. (we won't change this)
- `Jy` will be a new class. We'll implement it with autodiff. It'll accept the functor as an argument. It will expose one function that will be the implementation of Jy.
    ```C++
    template <class ode_rhs>
    struct Jy {
      std::vector<double> operator()(std::vector<double> y);
    };
    ```
- `Jtheta` will be a new class. We'll implement it with autodiff. It'll accept the functor as an argument. It will expose one function that will be the implementation of Jtheta.
    ```C++
    template <class ode_rhs>
    struct Jtheta {
      std::vector<double> operator()(std::vector<double> y);
    };
    ```
- `boost_ode_system` will replace `coupled_ode_system`. It will construct the appropriate functor that's required for the boost system. It'll figure out how to use Jy and Jtheta and when it actually needs them for solving ODEs. And how to construct the `stan::math::var`s with the appropriate gradients. This should look very similar to the coupled ode system that exists from the outside.
- `cvodes_ode_system` will now contain Jy and Jtheta, but not create the coupled ode system like `boost_ode_system`. Its job will be to construct these things into something that can be used by the `integrate_ode_bdf()` function. (We may need to rename this if it won't be applicable to all CVODES solvers.)

We need to expose indicators on Jy and Jtheta that let us know if autodiff is used. If we have those flags, we'll be able to parallelize at some point in the future.

By separating it out this way (Jy, Jtheta), a user may be able to provide analytic forms in C++.


# Open Issues

- How do we implement this? Someone needs to spec these out further with tests.

- Where do we stick the decoupling operation? Currently we have this being part of coupled_ode_system AND we have the function decouple_ode_states. As the *_ode_system concentrate on creating the right RHS to pass into the integrators, I think we should stick with the newer function decouple_ode_states, right? 

# Side notes

Help?! I'm out of time.


# Painless functional spec

I'm following this
http://www.joelonsoftware.com/articles/fog0000000035.html

Below this is the original doc written up by Sebastian.

# Dummy code

- usage of iterators of vectors to allow functions to "stream" their output into arbitrary locations in memory. In fact, actual implementations can assume a chunk of memory being allocated (and of the correct size as sensible by the context).

- I am now using a jacobian_ode function and Jy / Jtheta are different overloads. This felt more natural to write down than the class approach and looks elegant to me. So specifically the natural types determine what is being done here:
   - `Jy => jacobian_ode(const ode_rhs&, const vector<var>& y, const vector<double>& theta, vector<double>::iterator Jy)`
   - `Jtheta => jacobian_ode(const ode_rhs&, const vector<double>& y, const vector<var>& theta, vector<double>::iterator Jtheta)`

- I also introduce ode_rhs_sensitivity which does create the RHS of the sensitivities' states ODE

- As a convention I use y for states and yS for sensitivity states. Hence, we have ydot and ySdot which is the result of the RHS functions of each. So ydot = f(t,y,theta) and ySdot = fS(t,y,yS,theta). The f is represented by the ode_rhs, the fS by the ode_rhs_sensitivity.

- The ode_rhs_sensitivity object also holds the decouple_states operation and provides the initial_state of the sensitivity system.

- I changed boost_ode_system to odeint_ode_system since the library we are using is odeint which is part of boost (otherwise we would have to say sundials_ode_system to make it consistent instead of cvodes_ode_system).

```c++

// algo is generic for bdf or rk45
template <typename F, typename T_initial, typename T_param>
std::vector<std::vector<typename stan::return_type<T_initial,
                                                   T_param>::type> >
integrate_ode_algo(const F& f,
                   const std::vector<T_initial>& y0,
                   const double t0,
                   const std::vector<double>& ts,
                   const std::vector<T_param>& theta,
                   const std::vector<double>& x,
                   const std::vector<int>& x_int,
                   std::ostream* msgs = 0,
                   double relative_tolerance = 1e-10,
                   double absolute_tolerance = 1e-10,
                   long int max_num_steps = 1e8) {  // NOLINT(runtime/int)

  // make Stan system ready to provide needed stuff for integrator

  algo_ode_system<T_initial, T_param> ode_system(f, y0, theta, x, x_int, msgs);
  
  // setup integrator

  // somewhere ode_system is used which exposes the ode RHS and the
  // ode_sensitivity RHS separatley or as coupled system

  // solution states
  vector<vector<double> > y_coupled(T, vector<double>(N * (1+S))); 

  // do integration by looping over time points or letting the library do that, solution vector must endup in y_coupled

  algo_ode_system.decouple_states(y_coupled);
}


template <typename F, typename T_initial, typename T_param>
struct algo_ode_system : public ode_rhs_sensitivity<F, T_initial, T_param> {

  // algo specifc + init ode_rhs_sensitivity base
  
};

// example ode system for boost's odeint
template <typename F, typename T_initial, typename T_param>
struct odeint_ode_system : public ode_rhs_sensitivity<F, T_initial, T_param> {

  const size_t N_; // number of states

  /**
   * operator has the expected structure from odeint
   *
   * Assign the derivative vector with the system derivatives at
   * the specified state and time.
   *
   * <p>The input state must be of size <code>size()</code>, and
   * the output produced will be of the same size.
   *
   * @param[in] z state of the coupled ode system.
   * @param[out] dz_dt populated with the derivatives of
   * the coupled system at the specified state and time.
   * @param[in]  t time.
   *
   * y is the base ODE system state
   *
   */
  void operator()(const std::vector<double>& z,
                  std::vector<double>& dz_dt,
                  double t) const {
    // feed call into rhs sensitivity base class
    (*this)(t,
            z.cbegin(),  // pointer to the states y which are the first N entries
            z.cbegin() + N_, // points to the sensitivity states yS which are right after the N states
            dz_dt.begin(),  // the ydot goes into the front
            dz_dt.begin() + N_); // the ySdot follows this
  }
};

// this class represent fS and would need to be defined for the four cases
// d=double, v=var; first position is initial, second type of param;
// d,d = does essentially nothing as sensitivity system is empty
// v,d 
// d,v
// v,v
template <typename F, typename T_initial, typename T_param>
struct ode_rhs_sensitivity {

  // ode RHS
  const ode_rhs<F> rhs_;

  // initial state of the sensitivity system
  const vector<double> yS0_d_;

  const vector<T_param> theta_;
  
  // number of sensitivites
  const size_t S;
  // P=#of param, N=# of states
  // S=0 d,d
  // S=N v,d
  // S=P d,v
  // S=N+P v,v

  /**
   * Calculates the RHS of the sensitivity system.
   *
   * @param[in] t time
   * @param[in] y state at t
   * @param[in] yS state of sensitivity system at t (column major)
   * @param[out] ydot dy/dt (column major)
   * @param[out] ySdot dyS/dt (column major)
   */
  void operator()(double t,
                  vector<double>::const_iterator y,
                  vector<double>::const_iterator yS,
                  vector<double>::iterator ydot,
                  vector<double>::iterator ySdot
                  ) const {
    // use Jy
    vector<double> Jy(N_ * N_);
    const vector<var> y_v(y, y + N_);
    jacobian_ode(rhs_, y_v, theta_d_, ydot, Jy.begin());

    // and Jtheta 
    vector<double> Jtheta(N_ * S_);
    jacobian_ode(rhs_, y, theta_, ydot, Jtheta.begin());

    // use Eigen map facilities for expressive code or just hack it in
    // using for loops. Solution is streamed to ySdot.
  }

  // gets the solution vector, T x (N * (1+S))
  vector<vector<var or double> > decouple_states(vector<vector<double> >& states) const {}

  void initial_state(vector<double>::iterator yS0) const {
    // copy yS0_d to yS0
  };

  size_t size() const {
    return S;
  }
};


template<typename F>
struct ode_rhs {
  const vector<double> theta_d_;
  const F& f_;

  void operator()(double t, const vector<double>& y, vector<double>::iterator ydot) {
    // calls ODE RHS functor f_ with theta_d_ and copies results to ydot
  }

  // templated version to be used for AD
  template <typename T_state, typename T_param>
  std::vector<typename stan::return_type<T_state,
                                         T_param>::type>
  operator()(double t, const vector<T_state>& y, const vector<T_param>& theta) {
    // calls ODE RHS functor f_ with y and theta
  }
};

// THE jacobian_ode functions CAN BE SPECIALIZED TO BE ANALYTIC
// IMPLEMENTATIONS, this here corresponds to Jy
template<typename F>
void
jacobian_ode(const ode_rhs<F> rhs&,
             const vector<var>& y_v,
             const vector<double>& theta_d,
             vector<double>::iterator ydot,
             vector<double>::iterator Jy) {
  // use AD to calculate Jy and stream out to Jy. Will write NxN
  // elements (N=length of y_v vector)
}

// this one corresponds to Jtheta
template<typename F>
void
jacobian_ode(const ode_rhs<F>&,
             const vector<double>& y_d,
             const vector<var>& theta_v,
             vector<double>::iterator ydot,
             vector<double>::iterator Jtheta) {
  // use AD to calculate Jtheta
}

// optional (which we should add if calculating things in a single
// sweep is faster):
template<typename F>
void
jacobian_ode(const ode_rhs<F>&,
             const vector<var>& y_v,
             const vector<var>& theta_v,
             vector<double>::iterator ydot,
             vector<double>::iterator Jy,
             vector<double>::iterator Jtheta) {
  // use AD to calculate Jy and Jtheta, stream to Jy / Jtheta
}
```

-----------------------------------------------------



# Design Goals

In order of importance (open for discussion)

1. Unify non-stiff (rk45) and stiff (bdf) system
1. Have a `prim` version of integrator if possible
1. Enable convenient specification of analytic Jacobians instead of auto-diff; at this stage only via C++, enabling via the Stan language or other means at a later stage if feasible
1. Make objects passed to integrators free of var types with the goal to enable safe parallelization of ODE computations at some point.

# Requirements

For a mathematical description refer to

https://groups.google.com/group/stan-dev/attach/5b162997cf789/ode_system.pdf?part=0.1&authuser=0

# Definitions

Let's define our terms needed to set this up:

- t: time
- y: states at time t, dimensionality is N
- y0: initial state at t=t0 (also dim N)
- theta: parameters needed to calculate the ode_rhs, dimensionality is M
- ode_rhs: The basic ODE right-hand-side which calculates dy/dt = f(t,y,theta)
- Jy: Jacobian of the ode_rhs wrt to the states y
- Jtheta: Jacobian of the ode_rhs wrt to the parameters theta
- ode_rhs_sensitivity: Sensitivity RHS equations. These are derived from the ode_rhs using Jy and/or Jtheta. See the referred PDF above for details.

Note: Whenever parameters or initials are varying, the system gets enlarged by sensitivity variables, see the PDF above for details. Per sensitivity requested we hence gain N extra equations to solve. A convenient matrix way of expressing things is given in the PDF linked above.

# Current Designs

## Common Aspects

For both cases the ODE rhs equations and the ODE rhs sensitivity system is passed to the integrators. This is double only typed as the integrators are not aware of the Stan var system. The output from the integrators is also only double values and the output must be converted after integration to be usable as vars in the Stan system (using precomputed_gradients).

We have to deal with 4 different cases:

- dd: double y0 and double theta => needs no Jacobian
- dv: double y0 and var theta => needs Jy and Jtheta
- vd: var y0 and double theta => needs Jy
- vv: var y0 and var theta => needs Jy and Jtheta

## Non-Stiff RK45

- Based on boost odeint which expects the input function in form of a functor.
- The strategy is to combine the ode_rhs and the ode_rhs_sensitivity which yields the `coupled_ode_system`.
- Whenever initials vary the initial state y0 is set to 0 and a shifting operation is applied
- The `coupled_ode_system` includes the decoupling operation which converts the output from the integrator which include the solution of y and dy/dp at all requested times back into Stan usable var types using precomputed_gradients. During the decoupling the unshifting operation is applied.

integrate_ode_rk45 -> coupled_ode_system which discriminates the 4 different cases and provides the decouple_ode_operation

The coupled_ode_system internally calculates the sensitivity RHS directly in the specialized functor calls. **This is not aligned with the new requirement 3 above as one would need to specialize 3 different versions in case one wants to provide analytic Jacobians**.

## Stiff BDF

- Based on CVODES integrator which is specialised for solving ODEs with sensitivity support, needs three inputs:
    - ode_rhs: The base ODE only
    - ode_rhs_sensitivity: The ODE sensitivity RHS system only
    - Jy: for the BDF stiff integration the Jy is needed in addition (this is not needed should we use Adams-Moulton non-stiff at some point from CVODES)
- The strategy has been to introduce the `ode_system` object which provides
    - ode_rhs
    - jacobian function which gives Jy
    - jacobian function which givey Jy and Jtheta
- No shifting is performed, since setting the initial values of the sensitivity system differently makes shifting unnecessary.
- The decoupling operation has been split out and can now be applied after integration of the system which fulfills requirement 4.
- Implementation is based on the cvodes_user_data object which is a required object due to the static C callbacks. The callbacks are
    - `rhs`: ODE RHS, base system only
    - `rhs_sens`: ODE sensitivity RHS only
    - `dense_jacobian`: Jy needed when using stiff BDF integrator

# Proposed Changes

Pull requests could go like this:

1. Rename objects involved if deemed necessary
1. Refactor `coupled_ode_system`:
    1. Take decoupling operation out
    1. Do not do any shifting any more => change how initials are set
    1. Make coupled_ode_system use ode_system to get the Jacobians Jy and Jtheta
2. Refactor `cvodes_user_data`
    1. Make coupled_ode_system the base class of cvodes_ode_data
    2. Implement the `rhs_sens` callback by using the functor defined in cvodes_ode_data
    3. `ode_system` would still be needed in cvodes_user_data in order to get Jy for stiff integration
1. Optional: Further refactoring of integrate_ode_bdf and cvodes_ode_data with the goal to enable the Adams-Moulton integrator. To enable Adams-Moulton one only needs to initialize CVODES differently (~3 lines of code change)

This design would

- take away all Stan types out of objects being passed to the integrators
- have a single point where AD is done; in the ode_system object which then allows to plug in analytic Jacobians easily
- have a single point where the sensitivity system is defined (coupled_ode_system)