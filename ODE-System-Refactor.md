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
