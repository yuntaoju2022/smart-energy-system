# Using ESOPT

## Overview

`ESOPT` is a modeling and optimization platform built in C++. It provides standardized wrappers for commercial and open-source solvers, including `copt`, `cuopt`, `Cplex`, `Knitro`, `Uno`, `Ipopt`, `Gurobi`, `HiGHS`, and `MadNLP`, and offers a unified API for adding variables and constraints.

## Declaring a Model

Each solver is wrapped in a separate model class. Include the corresponding header file before declaring a model:

```cpp
#include "model_ipopt.h"
#include "model_highs.h"
#include "model_gurobi.h"
#include "model_knitro.h"
#include "model_copt.h"
#include "model_cplex.h"
#include "model_cuopt.h"
#include "model_mad.h"
#include "model_uno.h"

// Create instances
// Ipopt
IpoptModel ipopt_model;
// Highs
HighsModel highs_model;
// Gurobi
GurobiEnv env(true);
env.start();
GurobiModel gurobi_model(env);
// Knitro
KnitroModel knitro_model;
// Copt
COPTEnvConfig env_config;
COPTEnv env(env_config);
COPTModel model(env);
// Cplex
CplexEnv env();
env.start();
CplexModel gurobi_model(env);
// cuopt
CuoptModel model;
// MadNlp
MadModel model;
// Uno
UNOModel model;
```

All model classes derive from `ModelBase`, enabling polymorphic programming.

```cpp
#include "model_base.h"
ModelBase *model;
```

Every model must explicitly load the underlying solver's dynamic library. Load only the libraries required by your application.

```cpp
ipopt::load_library("path/to/the/Ipopt/dynamic/library");
highs::load_library("path/to/the/HiGHS/dynamic/library");
gurobi::load_library("path/to/the/Gurobi/dynamic/library");
knitro::load_library("path/to/the/Knitro/dynamic/library");
copt::load_library("path/to/the/copt/dynamic/library");
cplex::load_library("path/to/the/cplex/dynamic/library");
cuopt::load_library("path/to/the/cuopt/dynamic/library");
mad::load_library("path/to/the/mad/dynamic/library")
uno::load_library("path/to/the/uno/dynamic/library")
```

## Adding Variables

All models use `VariableIndex add_variable(double lb, double ub, VariableDomain domain = VariableDomain::Continuous, const char *name = nullptr)` to add a variable.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| lb | double | - | Lower bound of the variable. Must satisfy `lb <= ub`; use the infinity constant `-INFINITY` for no lower bound. |
| ub | double | - | Upper bound of the variable. Must satisfy `ub >= lb`; use the infinity constant `INFINITY` for no upper bound. |
| domain | VariableDomain | VariableDomain::Continuous | Variable-domain enumeration: `Continuous` for a continuous real-valued variable; `Integer` for an integer variable; `Binary` for a binary (0-1) variable. |
| name | const char* | nullptr | Variable name as a C string. When `nullptr` is passed, the model automatically generates an anonymous name. The function copies the string, so the caller may release its memory after the call. |

```cpp
/**
 * @brief Adds an optimization variable to the model.
 *
 * Creates an optimization variable with the specified bounds, domain, and name.
 * Returns an index handle for use when constructing constraints and objectives.
 *
 * @param lb Lower bound of the variable. Must satisfy lb <= ub.
 * @param ub Upper bound of the variable. Must satisfy ub >= lb.
 * @param domain Variable domain. Defaults to the continuous domain
 *               VariableDomain::Continuous. Available values are
 *               Continuous, Integer, and Binary.
 * @param name Variable name as a C string; may be nullptr. If nullptr is
 *             passed, an anonymous variable name is generated automatically.
 *             The function copies the string, so the caller may release its
 *             memory after the call.
 *
 * @return VariableIndex An index handle for the newly added variable.
 *         A valid handle can be used to access the variable's attributes and
 *         add constraints. An internal error throws an exception or returns
 *         an invalid index.
 *
 * @note
 * - After a successful call, the variable is registered with the model. The
 *   returned VariableIndex is valid only for the current model instance.
 * - lb and ub support infinite bounds. Use `INFINITY` or `-INFINITY` to create
 *   an unbounded side.
 * - Passing nullptr for name causes the model to generate an internal
 *   identifier, which does not affect the solution.
 *
 * @code
 * // Example 1: continuous variable with 0 <= x <= 10
 * auto x = model.add_variable(0.0, 10.0, VariableDomain::Continuous, "x");
 *
 * // Example 2: integer variable with 1 <= y <= 5 and no explicit name
 * auto y = model.add_variable(1.0,5.0, VariableDomain::Integer);
 *
 * // Example 3: binary variable
 * auto z = model.add_variable(0,1, VariableDomain::Binary, "z_bin");
 * @endcode
 */
VariableIndex add_variable(double lb, double ub, VariableDomain domain = VariableDomain::Continuous, const char *name = nullptr);
```

## Adding Constraints

### Linear Constraints

All models use `ConstraintIndex add_linear_constraint(const ScalarAffineFunction &f, ConstraintSense sense, double rhs, const char *name = nullptr)` to add a linear constraint.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| f | const ScalarAffineFunction & | - | Affine expression on the left-hand side of the constraint. It supports variable coefficients and a constant offset and may be used only for linear constraints. |
| sense | ConstraintSense | - | Constraint-sense enumeration: `LessEqual` for f <= rhs; `GreaterEqual` for f >= rhs; `Equal` for f = rhs. |
| rhs | double | - | Constant on the right-hand side of the equality or inequality. |
| name | const char* | nullptr | Constraint name as a C string. When `nullptr` is passed, an anonymous constraint is generated automatically. The function copies the string, so the caller may release its memory after the call. |

```cpp
/**
 * @brief Adds a linear constraint to the model.
 *
 * Constructs the linear constraint \f$ f \sim rhs \f$, where \f$f\f$ is a
 * scalar affine expression and the relation (<=, >=, or =) is specified by
 * sense. Registers the constraint with the optimization model and returns an
 * index handle that can subsequently be used to query its attributes.
 *
 * @param f ScalarAffineFunction representing the linear expression on the
 *          left-hand side, \(\sum a_i x_i + c\).
 * @param sense ConstraintSense specifying the comparison relation:
 *              - LessEqual: \(f \le rhs\)
 *              - GreaterEqual: \(f \ge rhs\)
 *              - Equal: \(f = rhs\)
 * @param rhs Constant on the right-hand side of the constraint.
 * @param name Constraint name as a C string. When nullptr is passed, the model
 *             automatically generates an anonymous name. The function copies
 *             the string, so the caller may safely release its memory after
 *             the call.
 *
 * @return ConstraintIndex An index handle for the newly added constraint.
 *         The handle is valid only for the current model instance and can be
 *         used to query constraint attributes.
 *
 * @note
 * - Only affine expressions are accepted. Passing an expression containing
 *   quadratic or nonlinear terms results in undefined behavior or an exception.
 * - ScalarAffineFunction may contain a constant offset, which is handled correctly.
 * - Constraints are not automatically simplified for redundancy. Duplicate
 *   constraints are added to the model unchanged.
 *
 * @code
 * // Example 1: x1 + 2*x2 <= 5
 * auto c1 = model.add_linear_constraint(x1 + 2*x2, ConstraintSense::LessThanOrEqual, 5.0, "c1");
 *
 * // Example 2: 3*x1 = 10, with no explicit name
 * auto c2 = model.add_linear_constraint(3*x1, ConstraintSense::Equal, 10.0);
 * @endcode
 */
ConstraintIndex add_linear_constraint(const ScalarAffineFunction &f, ConstraintSense sense, double rhs, const char *name = nullptr);
```

### Quadratic Constraints

All models that support quadratic constraints use `ConstraintIndex add_quadratic_constraint(const ScalarQuadraticFunction &f, ConstraintSense sense, double rhs, const char *name = nullptr)`.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| f | const ScalarQuadraticFunction & | - | Quadratic expression on the left-hand side of the constraint, including quadratic cross terms, linear terms, and a constant offset. |
| sense | ConstraintSense | - | Constraint-sense enumeration: `LessEqual` for f <= rhs; `GreaterEqual` for f >= rhs; `Equal` for f = rhs. |
| rhs | double | - | Constant on the right-hand side of the equality or inequality. |
| name | const char* | nullptr | Constraint name as a C string. When `nullptr` is passed, an anonymous constraint is generated automatically. The function copies the string, so the caller may release its memory after the call. |

```cpp
/**
 * @brief Adds a quadratic constraint to the model.
 *
 * Constructs the quadratic constraint \f$ f \sim rhs \f$, where \f$f\f$ is
 * a scalar quadratic function and the comparison relation is specified by
 * ConstraintSense. Registers the constraint with the optimization model and
 * returns an index handle that can subsequently be used to query its attributes.
 *
 * @param f ScalarQuadraticFunction representing the expression on the left-hand
 *          side, including quadratic terms, linear terms, and a constant offset.
 * @param sense ConstraintSense specifying the comparison relation:
 *              - LessEqual: \(f \le rhs\)
 *              - GreaterEqual: \(f \ge rhs\)
 *              - Equal: \(f = rhs\)
 * @param rhs Constant on the right-hand side of the constraint.
 * @param name Constraint name as a C string. When nullptr is passed, the model
 *             automatically generates an anonymous name. The function copies
 *             the string, so the caller may safely release its memory after
 *             the call.
 *
 * @return ConstraintIndex An index handle for the newly added constraint.
 *         The handle is valid only for the current model instance and can be
 *         used to query constraint attributes.
 *
 * @note
 * - The expression may contain quadratic terms, linear terms, and a constant
 *   offset. A purely linear expression is also accepted by this interface.
 * - The solver treats the constraint as quadratic. Even without quadratic
 *   terms, it is still processed through the quadratic-constraint path.
 * - Constraints are not automatically simplified for redundancy. Duplicate
 *   constraints are added to the model unchanged.
 *
 * @code
 * // Example 1: x1^2 + x2 <= 10
 * auto c1 = model.add_quadratic_constraint(x1*x1+x2, ConstraintSense::LessEqual, 10.0, "qc1");
 *
 * // Example 2: x1*x2 = 2.0, as an anonymous quadratic constraint
 * auto c2 = model.add_quadratic_constraint(x1*x2, ConstraintSense::Equal, 2.0);
 * @endcode
 */
ConstraintIndex add_quadratic_constraint(const ScalarQuadraticFunction &f, ConstraintSense sense, double rhs, const char *name = nullptr);
```

### Nonlinear Constraints

Models that support nonlinear constraints use `NLConstraintIndex add_nl_constraint(const ExpressionHandle &expr, ConstraintSense sense, double rhs, const char *name = nullptr)`.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| expr | const ExpressionHandle & | - | Handle to the expression on the left-hand side. Supports linear and quadratic expressions as well as nonlinear operations such as exp, sin, cos, and pow. The expression must be differentiable. |
| sense | ConstraintSense | - | Constraint-sense enumeration: `LessEqual` for expr <= rhs; `GreaterEqual` for expr >= rhs; `Equal` for expr = rhs. |
| rhs | double | - | Constant on the right-hand side of the equality or inequality. |
| name | const char* | nullptr | Constraint name as a C string. When `nullptr` is passed, an anonymous constraint is generated automatically. The function copies the string, so the caller may release its memory after the call. |

```cpp
/**
 * @brief Adds a general nonlinear constraint.
 *
 * Constructs the nonlinear constraint \f$ expr \sim rhs \f$. The left-hand
 * side is a general ExpressionHandle supporting linear, quadratic, and any
 * differentiable nonlinear operations. Registers the constraint with the model
 * and returns a nonlinear-constraint index for subsequent attribute queries.
 *
 * @attention
 * <b>RAII requirements for the expression context (important)</b>
 * Nonlinear expressions (sin, cos, tan, pow, and so on) must be created under
 * the management of an @c ExpressionGraphContextGuard RAII guard. Every
 * @c ExpressionHandle remains valid only while that guard object is alive.
 * This function must be called to add the constraint before the guard is
 * destroyed. Once the guard leaves scope, every ExpressionHandle created in
 * that scope becomes invalid. Reusing those handles results in dangling
 * references and undefined behavior. A single guard scope may be used to build
 * multiple expressions and call add_nl_constraint multiple times.
 *
 * @param expr Handle to the expression on the left-hand side of the constraint.
 *             Supports linear, quadratic, exponential, trigonometric, power,
 *             and other expressions. The expression must be differentiable;
 *             a nondifferentiable expression may cause Ipopt to fail.
 * @param sense @c ConstraintSense specifying the comparison relation:
 *              - LessEqual: \f$ expr \le rhs \f$
 *              - GreaterEqual: \f$ expr \ge rhs \f$
 *              - Equal: \f$ expr = rhs \f$
 * @param rhs Constant on the right-hand side of the constraint.
 * @param name Constraint name as a C string. When @c nullptr is passed, the
 *             model automatically generates an anonymous name. The function
 *             copies the string, so the caller may safely release its memory
 *             after the call.
 *
 * @return NLConstraintIndex An index handle for the newly added nonlinear
 *         constraint. The handle is valid only for the current model instance
 *         and can be used to access constraint attributes.
 *
 * @note
 * - This is the general nonlinear interface. Although it accepts linear and
 *   quadratic expressions, the specialized interfaces provide better
 *   performance. Prefer @ref add_linear_constraint() for linear constraints
 *   and @ref add_quadratic_constraint() for quadratic constraints.
 * - An @c NLConstraintIndex returned by one model instance must not be used
 *   with another model instance.
 *
 * @code
 * // Correct: build expressions and add constraints within an RAII guard scope
 * {
 *     ExpressionGraphContextGuard guard; // Activates the expression-graph context
 *     auto expr1 = sin(x1 + x1 * x2);
 *     model.add_nl_constraint(expr1, ConstraintSense::LessEqual, 5.0, "nl_con1");
 *
 *     auto expr2 = cos(x1 + x1 * x2);
 *     model.add_nl_constraint(expr2, ConstraintSense::LessEqual, 5.0, "nl_con2");
 *
 *     auto expr3 = tan(x1 + x1 * x2);
 *     model.add_nl_constraint(expr3, ConstraintSense::LessEqual, 5.0, "nl_con3");
 * }
 * // The guard has left scope; expr1, expr2, and expr3 are now invalid.
 *
 * @endcode
 */
NLConstraintIndex add_nl_constraint(const ExpressionHandle &expr, ConstraintSense sense, double rhs, const char *name = nullptr);
```

## Objective Functions

### Linear and Quadratic Objective Functions

Use the following function to set a linear or quadratic objective:

`void set_objective(const ExprBuilder &expr, ObjectiveSense sense);`

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| expr | const ExprBuilder & | - | Objective-expression builder. Accepts a linear `ScalarAffineFunction` or quadratic `ScalarQuadraticFunction`; general nonlinear expressions such as sin, cos, and exp are not supported. |
| sense | ObjectiveSense | - | Optimization-direction enumeration: `Minimize` minimizes the objective; `Maximize` maximizes it. |

```cpp
/**
 * @brief Sets the objective function for the optimization problem.
 *
 * Sets the model's objective function and optimization direction. The
 * ExprBuilder argument supports linear and quadratic expressions but does not
 * accept general nonlinear expressions.
 *
 * @param expr Objective-expression builder @c ExprBuilder. Supports linear and
 *             quadratic expressions; general nonlinear terms such as sin,
 *             cos, and exp are not supported.
 * @param sense Optimization direction:
 *              - ObjectiveSense::Minimize: minimize the objective function
 *              - ObjectiveSense::Maximize: maximize the objective function
 *
 * @return void
 *
 * @attention
 * - If set_objective is called multiple times, each call replaces the previous
 *   objective. The model retains only the most recently specified objective.
 * - This interface supports only linear and quadratic objectives. Passing an
 *   expression containing nonlinear operators results in an exception or
 *   undefined behavior.
 *
 * @code
 * // Example 1: set the linear minimization objective min 2*x1 + 3*x2
 * model.set_objective(2*x1 + 3*x2, ObjectiveSense::Minimize);
 *
 * // Example 2: set the quadratic maximization objective max x1*x1 + 2*x2
 * model.set_objective(x1*x1 + 2*x2, ObjectiveSense::Maximize);
 * @endcode
 */
void set_objective(const ExprBuilder &expr, ObjectiveSense sense);
```

### Nonlinear Objective Functions

When a model's objective is a nonlinear expression, use:

`void add_nl_objective(const ExpressionHandle &expr);`

Currently, only **minimization** is supported.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| expr | const ExpressionHandle & | - | Handle to a nonlinear objective term. It must be constructed while an `ExpressionGraphContextGuard` is alive. Supports differentiable nonlinear operations such as sin, cos, exp, and pow. Where required, convert a `VariableIndex` to an expression handle with `expr(var)` before using the variable in an expression. The expression must be differentiable. |

```cpp
/**
 * @brief Adds a nonlinear objective term.
 *
 * Appends a nonlinear expression to the model's objective function.
 *
 * @attention
 * <b>RAII requirements for the expression context (important)</b>
 * The supplied @c ExpressionHandle must be constructed while an
 * @c ExpressionGraphContextGuard RAII guard is alive. This function must be
 * called before the guard is destroyed. Once the guard leaves scope, all
 * @c ExpressionHandle objects created in that scope become invalid. Reusing
 * those handles results in dangling references and undefined behavior.
 *
 * - In some expressions, a variable must be explicitly converted from
 *   VariableIndex to ExpressionHandle with @c expr(var).
 *
 * @param expr Handle to a nonlinear objective expression. Supports
 *             differentiable nonlinear operations such as sin, cos, tan, exp,
 *             log, and pow, as well as linear and quadratic expressions.
 *
 * @return void
 *
 * @code
 * // Correct: build and add nonlinear objective expressions under an RAII guard
 * {
 *     ExpressionGraphContextGuard guard_con1;
 *
 *     // Simple nonlinear objective term with trigonometric functions
 *     model.add_nl_objective( sin( expr(x1) + expr(x2) ) );
 *
 *     // Composite expression with variables converted using expr()
 *     model.add_nl_objective( expr(x1) * expr(x4) * (expr(x1)+expr(x2)+expr(x3)) + expr(x3) );
 * }
 * // guard_con1 is destroyed. Local ExpressionHandle objects are invalid, but
 * // the nonlinear objective terms already added to the model remain valid.
 *
 * @endcode
 */
void add_nl_objective(const ExpressionHandle &expr);
```

## Solving a Model

Use the following function to solve a model:

```cpp
void optimize();
```

```cpp
/**
 * @brief Runs the optimization by invoking the solver on the current model.
 *
 * Submits the variables, constraints, and objective function in the completed
 * model to the solver for numerical optimization. After the solve finishes,
 * variable values are stored in the model and can be retrieved through the API,
 * together with information such as constraint dual values.
 *
 * @return void
 *
 * @attention
 * - Before calling @c optimize(), complete the required model configuration:
 *   1. Add at least one optimization variable.
 *   2. Set the objective and optimization direction with @ref set_objective().
 *   3. Add linear, quadratic, or nonlinear constraints and nonlinear objective
 *      terms as needed.
 * - If the solve fails, the model retains the status information returned by
 *   the solver. Use the status-query interface to obtain the reason for failure.
 *
 * @note
 * - This function blocks until the solve is complete.
 * - After the solve, the solution for a VariableIndex can be retrieved.
 * - optimize() may be called multiple times on the same model instance. Each
 *   call starts a new solve.
 * - @c ExpressionGraphContextGuard does not affect this interface. The solve
 *   phase no longer depends on the expression-stack RAII context.
 *
 * @code
 * // Complete solution workflow
 * // 1. Add variables
 * auto x1 = model.add_variable(0.0, 10.0, VariableDomain::Continuous, "x1");
 * auto x2 = model.add_variable(0.0, 10.0, VariableDomain::Continuous, "x2");
 *
 * // 2. Set the objective function
 * model.set_objective(x1+x2, ObjectiveSense::Minimize);
 *
 * // 3. Add a constraint
 * {
 *     ExpressionGraphContextGuard guard;
 *     model.add_nl_constraint(expr(x1)+expr(x2), ConstraintSense::LessEqual, 8.0);
 * }
 *
 * // 4. Run the solve (blocking call)
 * model.optimize();
 *
 * // 5. Retrieve the variable value
 * double val = model.get_variable_value(x1);
 * @endcode
 */
void optimize();
```

## Setting Solver Parameters

Use the following overloads to set solver parameters:

```cpp
/**
 * @brief Sets an Ipopt, Gurobi, HiGHS, or other solver option by name.
 *
 * These overloaded interfaces set floating-point, integer, and string solver
 * parameters. Parameter names are identical to the solver's native option
 * names. The settings take effect during a subsequent @ref optimize() call.
 *
 * @param name Solver option name as a C string. It must exactly match the
 *             official option name and is case-sensitive.
 * @param value Option value of type double.
 *
 * @return void
 *
 * @attention
 * - Call these interfaces **before** @ref optimize(). Parameters set after the
 *   solve begins do not affect the current solve.
 * - Parameter names must follow the solver's official option definitions.
 *   An incorrect name or mismatched type does not produce an error.
 * - When the same name is set multiple times, the latest value replaces the
 *   previous value.
 *
 * @note
 * - The three overloads accept double, int, and const char*, respectively.
 *   Choose the overload that matches the parameter's actual type.
 * - Consult the solver's official documentation for all supported parameter
 *   names and value ranges.
 *
 * @code
 * // Set a floating-point parameter: convergence tolerance
 * model.set_parameter("tol", 1e-6);
 *
 * // Set an integer parameter: maximum number of iterations
 * model.set_parameter("max_iter", 1000);
 *
 * // Set a string parameter: output log level
 * model.set_parameter("print_level", "5");
 * @endcode
 */
void set_parameter(const char *name, double value);

/**
 * @brief Sets a solver option by name (integer overload).
 *
 * @param name Solver option name as a C string. It must exactly match the
 *             official option name and is case-sensitive.
 * @param value Option value of type int.
 *
 * @return void
 *
 * @sa set_parameter(const char *, double)
 */
void set_parameter(const char *name, int value);

/**
 * @brief Sets a solver option by name (string overload).
 *
 * @param name Solver option name as a C string. It must exactly match the
 *             official option name and is case-sensitive.
 * @param value Option value as a string. The function copies the string, so the
 *              caller may release its memory after the call.
 *
 * @return void
 *
 * @sa set_parameter(const char *, double)
 */
void set_parameter(const char *name, const char *value);
```

### Retrieving Variable Attributes

Use the following interfaces to retrieve variable attributes:

```cpp
double get_variable_attribute(const VariableIndex& variable, VariableAttributeDouble attribute);
int get_variable_attribute(const VariableIndex& variable, VariableAttributeInt attribute) ;
std::string get_variable_attribute(const VariableIndex& variable, VariableAttributeString attribute) override;
```

```cpp
/**
 * @brief Retrieves a floating-point variable attribute.
 *
 * This general interface reads a floating-point attribute of the specified
 * variable while abstracting the underlying solver. The attribute is selected
 * by the VariableAttributeDouble enumeration.
 *
 * @param variable VariableIndex handle for the target variable. It must belong
 *                 to the current model instance.
 * @param attribute VariableAttributeDouble value identifying the attribute to
 *                  read, such as the lower bound, upper bound, or optimal value.
 *
 * @return double The value of the requested attribute.
 *
 * @attention
 * - Some attributes, such as the optimal value, are valid only after
 *   @ref optimize() has completed. Reading them before solving may return an
 *   invalid value.
 *
 * @sa VariableAttributeDouble
 * @sa get_variable_attribute(const VariableIndex&, VariableAttributeInt)
 * @sa get_variable_attribute(const VariableIndex&, VariableAttributeString)
 *
 * @code
 * // Retrieve the variable's optimal value after solving
 * double x_opt = model.get_variable_attribute(x, VariableAttributeDouble::PrimalValue);
 * // Retrieve the variable's lower bound
 * double lb = model.get_variable_attribute(x, VariableAttributeDouble::LowerBound);
 * @endcode
 */
double get_variable_attribute(const VariableIndex& variable, VariableAttributeDouble attribute);

/**
 * @brief Retrieves an integer variable attribute.
 *
 * This general interface reads an integer attribute of the specified variable
 * while abstracting the underlying solver. It is not limited to Ipopt. The
 * attribute is selected by the VariableAttributeInt enumeration.
 *
 * @param variable VariableIndex handle for the target variable. It must belong
 *                 to the current model instance.
 * @param attribute VariableAttributeInt value identifying the attribute to read,
 *                  such as the variable-domain identifier or a solution-status flag.
 *
 * @return int The value of the requested attribute.
 *
 * @attention
 * - Some attributes are valid only after @ref optimize() has completed.
 * - Attribute support depends on the backend solver.
 *
 * @sa VariableAttributeInt
 */
int get_variable_attribute(const VariableIndex& variable, VariableAttributeInt attribute) ;

/**
 * @brief Retrieves a string variable attribute.
 *
 * This general interface reads a string attribute, such as a variable name,
 * while abstracting the underlying solver. The attribute is selected by the
 * VariableAttributeString enumeration.
 *
 * @param variable VariableIndex handle for the target variable. It must belong
 *                 to the current model instance.
 * @param attribute VariableAttributeString value identifying the attribute to
 *                  read, such as the variable name.
 *
 * @return std::string The requested attribute. Returns an empty string if the
 *         variable has no name.
 *
 * @attention
 * - Attribute support depends on the backend solver.
 *
 * @sa VariableAttributeString
 *
 * @code
 * std::string var_name = model.get_variable_attribute(x, VariableAttributeString::Name);
 * @endcode
 */
std::string get_variable_attribute(const VariableIndex& variable, VariableAttributeString attribute);
```

## Attributes

### Constraint Sense

```cpp
/**
 * @brief Enumeration of constraint comparison relations.
 *
 * Defines the comparison between the expression on the left-hand side of a
 * linear or quadratic constraint and its right-hand side, rhs.
 */
enum class ConstraintSense
{
    /// \f$ f \le rhs \f$: less than or equal to
    LessEqual,
    /// \f$ f \ge rhs \f$: greater than or equal to
    GreaterEqual,
    /// \f$ f = rhs \f$: equal to
    Equal,
};
```

### Objective Direction

```cpp
/**
 * @brief Enumeration of objective-function optimization directions.
 */
enum class ObjectiveSense
{
    /// Minimize the objective function
    Minimize,
    /// Maximize the objective function
    Maximize
};
```

### Variable Attributes

```cpp
enum class VariableAttributeDouble
{
    Value,
    LowerBound,
    UpperBound,
    PrimalStart,
};


enum class VariableAttributeString
{
    Domain,
    Name,
};
```
