# ESOPT使用

## 概述

`ESOPT`是使用C++构建的建模求解平台，设计标准接口封装了包括`copt`、`cuopt`、`Cplex`、`Knitro`、`Uno`、`Ipopt`、`Gurobi`、`HiGHS`、`MadNLP`等商业和开源求解器，使用统一的API添加变量和约束。

## 模型声明

不同的求解器封装在不同的模型类中，声明时需要先导入对应的头文件：

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

# 创建实例
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

所有的模型类都派生于`ModelBase`，可以实现多态编程。

```cpp
#include "model_base.h"
ModelBase *model;
```

所有模型需要主动导入底层求解器的动态库，根据需要按需导入。

```cpp
ipopt::load_library("Ipopt的动态库路径");
highs::load_library("HiGHS的动态库路径");
gurobi::load_library("Gurobi的动态库路径");
knitro::load_library("Knitro的动态库路径");
copt::load_library("copt的动态库路径");
cplex::load_library("cplex的动态库路径");
cuopt::load_library("cuopt的动态库路径");
mad::load_library("mad的动态库路径")
uno::load_library("uno的动态库路径")
```

## 添加变量

所有模型统一使用`VariableIndex add_variable(double lb, double ub, VariableDomain domain = VariableDomain::Continuous, const char *name = nullptr)`添加变量到模型中.

| 参数名 | 类型           | 默认值                     | 说明                                                         |
| ------ | -------------- | -------------------------- | ------------------------------------------------------------ |
| lb     | double         | —                          | 变量下界。必须满足 `lb <= ub`；可使用  -INFINITY无穷常量表示无下界。 |
| ub     | double         | —                          | 变量上界。必须满足 `ub >= lb`；可使用 INFINITY无穷常量表示无上界。 |
| domain | VariableDomain | VariableDomain::Continuous | 变量域枚举类型：- `Continuous`：连续实数变量- `Integer`：整数变量- `Binary`：0‑1 二元变量 |
| name   | const char*    | nullptr                    | 变量名称 C 字符串。传入`nullptr`时模型自动生成匿名名称；函数内部复制字符串内容，外部传入字符串可调用后释放。 |

```cpp
/**
 * @brief 向模型中添加一个优化变量
 *
 * 创建一个新的优化变量，设置上下界、变量类型、变量名称，返回该变量的索引句柄，用于后续约束、目标函数构建。
 *
 * @param lb 变量下界(lower bound)。注意数值合法性：lb 应当 <= ub。
 * @param ub 变量上界(upper bound)。注意数值合法性：ub 应当 >= lb。
 * @param domain 变量域类型，默认为连续变量 VariableDomain::Continuous。
 *               可选枚举：Continuous(连续) / Integer(整数) / Binary(0‑1二元)。
 * @param name 变量名称，C风格字符串，可为nullptr；传入nullptr时自动生成匿名变量名。
 *             指针仅使用字符串内容，函数内部会做拷贝，外部传入的字符串内存可在调用后释放。
 *
 * @return VariableIndex 返回该新增变量的索引句柄。
 *         返回值为有效句柄，可用于访问该变量的属性、添加约束；
 *         若内部异常，会抛出异常/返回非法索引
 *
 * @note
 * - 调用成功后，变量注册到模型内部；返回的 VariableIndex 仅对当前模型实例有效。
 * - lb、ub 支持无穷边界，可使用 `INFINITY` / `-INFINITY` 设置无界。
 * - name 传入 nullptr，模型自动生成内部标识名称，不影响求解。
 *
 * @code
 * // 示例1：普通连续变量 0 <= x <= 10
 * auto x = model.add_variable(0.0, 10.0, VariableDomain::Continuous, "x");
 *
 * // 示例2：整数变量 1 <= y <=5，不指定名字
 * auto y = model.add_variable(1.0,5.0, VariableDomain::Integer);
 *
 * // 示例3：二元0‑1变量
 * auto z = model.add_variable(0,1, VariableDomain::Binary, "z_bin");
 * @endcode
 */
VariableIndex add_variable(double lb, double ub, VariableDomain domain = VariableDomain::Continuous, const char *name = nullptr);
```

## 添加约束

### 线性约束

所有模型统一使用`ConstraintIndex add_linear_constraint(const ScalarAffineFunction &f, ConstraintSense sense, double rhs, const char *name = nullptr)`

| 参数名 | 类型                         | 默认值  | 说明                                                         |
| ------ | ---------------------------- | ------- | ------------------------------------------------------------ |
| f      | const ScalarAffineFunction & | —       | 约束左侧仿射线性表达式，支持变量系数 + 常数偏移，只能用于线性约束。 |
| sense  | ConstraintSense              | —       | 约束比较方向枚举：`LessEqual`：f≤rhs；`GreaterEqual`：f≥rhs；`Equal`：f=rhs |
| rhs    | double                       | —       | 约束等式 / 不等式右侧常数。                                  |
| name   | const char*                  | nullptr | 约束名称 C 字符串。传入`nullptr`自动生成匿名约束；函数内部复制字符串，外部内存调用后可释放。 |

```cpp
/**
 * @brief 向模型添加一条线性约束
 *
 * 构造线性约束 \f$ f \sim rhs \f$，其中 \f$f\f$ 为标量仿射表达式，~由约束sense指定(<=, >=, =)。
 * 将约束注册到优化模型内部，返回约束索引句柄，后续可通过该索引查询约束属性。
 *
 * @param f 标量仿射函数 ScalarAffineFunction，代表约束左侧线性表达式 \(\sum a_i x_i + c\)。
 * @param sense 约束方向枚举 ConstraintSense，指定约束比较关系：
 *              - LessEqual：\(f \le rhs\)
 *              - GreaterEqual：\(f \ge rhs\)
 *              - Equal：\(f = rhs\)
 * @param rhs 约束右侧常数项。
 * @param name 约束名称，C风格字符串；传 nullptr 时模型自动生成匿名约束名。
 *             函数内部拷贝字符串内容，外部传入字符串内存调用后可安全释放。
 *
 * @return ConstraintIndex 返回新增约束的索引句柄。
 *         句柄仅对当前模型实例有效，用于查询约束属性；
 *         
 *
 * @note
 * - 仅接受线性仿射表达式；传入含二次/非线性项的表达式会产生未定义行为或抛出异常。
 * - ScalarAffineFunction 允许带有常数偏移项，内部会正确处理。
 * - 约束不会自动做冗余化简，重复约束会原样加入模型。
 *
 * @code
 * // 示例1： x1 + 2*x2 <= 5

 * auto c1 = model.add_linear_constraint(x1 + 2*x2, ConstraintSense::LessThanOrEqual, 5.0, "c1");
 *
 * // 示例2： 3*x1 = 10，不指定名称
 * auto c2 = model.add_linear_constraint(3*x1, ConstraintSense::Equal, 10.0);
 * @endcode
 */
ConstraintIndex add_linear_constraint(const ScalarAffineFunction &f, ConstraintSense sense, double rhs, const char *name = nullptr);
```

### 二次约束

所有模型（支持二次约束）统一使用`ConstraintIndex add_quadratic_constraint(const ScalarQuadraticFunction &f, ConstraintSense sense, double rhs, const char *name = nullptr)`。

| 参数名 | 类型                            | 默认值  | 说明                                                         |
| ------ | ------------------------------- | ------- | ------------------------------------------------------------ |
| f      | const ScalarQuadraticFunction & | —       | 约束左侧二次表达式，包含二次交叉项、一次项、常数偏移。       |
| sense  | ConstraintSense                 | —       | 约束比较方向枚举：`LessEqual`：f≤rhs；`GreaterEqual`f≥rhs；`Equal`：f=rhs |
| rhs    | double                          | —       | 约束等式 / 不等式右侧常数。                                  |
| name   | const char*                     | nullptr | 约束名称 C 字符串。传入`nullptr`自动生成匿名约束；函数内部复制字符串，外部内存调用后可释放。 |

```cpp
/**
 * @brief 向模型添加一条二次约束
 *
 * 构造二次约束 \f$ f \sim rhs \f$，其中 \f$f\f$ 为标量二次函数，比较关系由 ConstraintSense 指定。
 * 将约束注册到优化模型内部，返回约束索引句柄，后续可通过该索引查询约束属性。
 *
 * @param f 标量二次函数 ScalarQuadraticFunction，代表约束左侧表达式，包含二次项、一次项与常数偏移项。
 * @param sense 约束方向枚举 ConstraintSense，指定约束比较关系：
 *              - LessEqual：\(f \le rhs\)
 *              - GreaterEqual：\(f \ge rhs\)
 *              - Equal：\(f = rhs\)
 * @param rhs 约束右侧常数项。
 * @param name 约束名称，C风格字符串；传 nullptr 时模型自动生成匿名约束名。
 *             函数内部拷贝字符串内容，外部传入字符串内存调用后可安全释放。
 *
 * @return ConstraintIndex 返回新增约束的索引句柄。
 *         句柄仅对当前模型实例有效，用于查询约束属性；
 *
 *
 * @note
 * - 表达式允许同时存在二次项、一次项、常数偏移；纯线性表达式也可使用本接口。
 * - 求解器层面视该约束为二次约束；即使无二次项，仍会按二次约束路径处理。
 * - 约束不会自动做冗余化简，重复约束会原样加入模型。
 *
 * @code
 * // 示例1： x1^2 + x2 <= 10
 * auto c1 = model.add_quadratic_constraint(x1*x1+x2, ConstraintSense::LessEqual, 10.0, "qc1");
 *
 * // 示例2： x1*x2 = 2.0，匿名二次约束
 * auto c2 = model.add_quadratic_constraint(x1*x2, ConstraintSense::Equal, 2.0);
 * @endcode
 */
ConstraintIndex add_quadratic_constraint(const ScalarQuadraticFunction &f, ConstraintSense sense, double rhs, const char *name = nullptr);
```

### 非线性约束

所有支持非线性约束的模型，使用`NLConstraintIndex add_nl_constraint(const ExpressionHandle &expr, ConstraintSense sense, double rhs, const char *name = nullptr)`添加非线性约束。

| 参数名 | 类型                     | 默认值  | 说明                                                         |
| ------ | ------------------------ | ------- | ------------------------------------------------------------ |
| expr   | const ExpressionHandle & | —       | 约束左侧表达式句柄，支持线性、二次、exp/sin/cos/pow 等非线性运算；表达式必须可微。 |
| sense  | ConstraintSense          | —       | 约束比较方向枚举：`LessEqual`：expr≤rhs`GreaterEqual`：expr≥rhs`Equal`：expr=rhs |
| rhs    | double                   | —       | 约束等式 / 不等式右侧常数右端项。                            |
| name   | const char*              | nullptr | 约束名称 C 字符串。传入`nullptr`自动生成匿名约束；函数内部复制字符串，外部内存调用后可释放。 |

```cpp
/**
 * @brief 添加通用非线性约束
 *
 * 构建非线性约束 \f$ expr \sim rhs \f$。左侧为通用表达式句柄 ExpressionHandle，支持线性、二次以及任意可微非线性运算。
 * 将约束注册至模型，返回非线性约束索引句柄，可用于后续查询该约束属性。
 *
 * @attention
 * <b>表达式上下文 RAII 约束（重要）</b>
 * 创建非线性表达式（sin/cos/tan/pow等）必须受 @c ExpressionGraphContextGuard RAII守卫管理。
 * 所有表达式句柄 @c ExpressionHandle 仅在该守卫对象的存活作用域内有效。
 * 必须在 guard 对象未析构之前调用本函数完成约束添加；guard出作用域析构后，所有该域生成的ExpressionHandle全部失效，
 * 禁止再使用这些expr句柄，否则产生悬空引用、未定义行为。
 * 一个 Guard 作用域内可以构造多条表达式、多次调用 add_nl_constraint。
 *
 * @param expr 约束左侧表达式句柄 @c ExpressionHandle。
 *             支持线性、二次、指数、三角函数、幂函数等各类表达式；
 *             表达式需要可微，不可微表达式会导致Ipopt求解失败。
 * @param sense 约束比较关系枚举 @c ConstraintSense：
 *              - LessEqual：\f$ expr \le rhs \f$
 *              - GreaterEqual：\f$ expr \ge rhs \f$
 *              - Equal：\f$ expr = rhs \f$
 * @param rhs 约束右侧常数右端项。
 * @param name 约束名称，C风格字符串；传入 @c nullptr 时模型自动生成匿名约束名称。
 *             函数内部复制字符串内容，调用结束后外部传入的字符串内存可安全释放。
 *
 * @return NLConstraintIndex 新增非线性约束的索引句柄。
 *         句柄仅在当前模型实例下有效，用于访问约束属性；
 *
 * @note
 * - 本接口是通用非线性入口，虽然可以传入线性/二次表达式，但性能不如专用接口；
 *   线性约束优先使用 @ref add_linear_constraint()，二次约束优先使用 @ref add_quadratic_constraint()。
 * - 返回的 @c NLConstraintIndex 禁止跨不同模型实例混用。
 *
 * @code
 * // 正确示例：RAII守卫作用域内构造表达式并添加约束
 * {
 *     ExpressionGraphContextGuard guard; // 激活表达式图上下文，RAII守卫
 *     auto expr1 = sin(x1 + x1 * x2);
 *     model.add_nl_constraint(expr1, ConstraintSense::LessEqual, 5.0, "nl_con1");
 *
 *     auto expr2 = cos(x1 + x1 * x2);
 *     model.add_nl_constraint(expr2, ConstraintSense::LessEqual, 5.0, "nl_con2");
 *
 *     auto expr3 = tan(x1 + x1 * x2);
 *     model.add_nl_constraint(expr3, ConstraintSense::LessEqual, 5.0, "nl_con3");
 * }
 * // guard离开作用域析构：expr1/expr2/expr3全部失效；
 *
 * @endcode
 */
NLConstraintIndex add_nl_constraint(const ExpressionHandle &expr, ConstraintSense sense, double rhs, const char *name = nullptr);
```

## 目标函数

### 线性和二次目标函数

线性或者二次目标函数设置使用

`void set_objective(const ExprBuilder &expr, ObjectiveSense sense);`

| 参数名 | 类型                | 默认值 | 说明                                                         |
| ------ | ------------------- | ------ | ------------------------------------------------------------ |
| expr   | const ExprBuilder & | —      | 目标函数表达式构建器，可传入线性`ScalarAffineFunction`或二次`ScalarQuadraticFunction`；不支持 sin/cos/exp 等通用非线性表达式。 |
| sense  | ObjectiveSense      | —      | 优化方向枚举：`Minimize`：最小化目标, `Maximize`：最大化目标 |

```cpp
/**
 * @brief 设置优化问题目标函数
 *
 * 设置模型的目标函数与优化方向。左侧接收表达式构建器 ExprBuilder，
 * 支持线性表达式、二次表达式，用于构造目标；不接收通用非线性表达式，
 *
 * @param expr 目标函数表达式构建器 @c ExprBuilder。
 *             支持线性、二次表达式；不支持sin/cos/exp等一般非线性项。
 * @param sense 目标优化方向：
 *              - ObjectiveSense::Minimize：最小化目标函数
 *              - ObjectiveSense::Maximize：最大化目标函数
 *
 * @return void 无返回值。
 *
 * @attention
 * - 调用多次 set_objective：后调用会覆盖之前设置的目标函数，模型只保留最后一次设置的目标。
 * - 本接口仅支持线性、二次目标；若传入包含非线性算子的表达式，会触发异常或未定义行为。
 *
 *
 * @code
 * // 示例1：设置线性最小化目标 min 2*x1 + 3*x2
 * model.set_objective(2*x1 + 3*x2, ObjectiveSense::Minimize);
 *
 * // 示例2：设置二次最大化目标 max x1*x1 + 2*x2
 * model.set_objective(x1*x1 + 2*x2, ObjectiveSense::Maximize);
 * @endcode
 */
void set_objective(const ExprBuilder &expr, ObjectiveSense sense);
```

### 非线性目标函数

模型的目标函数是非线性表达式时，使用

`void add_nl_objective(const ExpressionHandle &expr);`

当前只能求解**最小**。

| 参数名 | 类型                     | 默认值 | 说明                                                         |
| ------ | ------------------------ | ------ | ------------------------------------------------------------ |
| expr   | const ExpressionHandle & | —      | 非线性目标项表达式句柄。必须在`ExpressionGraphContextGuard`存活域内构造；支持 sin/cos/exp/pow 等可微非线性运算；变量参与运算必要时使用`expr(var)`将`VariableIndex`转为表达式句柄；表达式必须可微。 |

```cpp
/**
 * @brief 添加非线性目标项
 *
 * 向模型目标函数追加一项非线性表达式。
 *
 * @attention
 * <b>表达式上下文 RAII 约束（重要）</b>
 * 传入的 @c ExpressionHandle 必须在 @c ExpressionGraphContextGuard RAII守卫的存活作用域内构造。
 * 需要在 guard 对象未析构前完成本函数调用；guard离开作用域析构后，该域生成的全部 @c ExpressionHandle 失效，
 * 禁止再使用这些句柄，否则产生悬空引用、未定义行为。
 *
 * - 变量直接参与表达式运算时，部分场景需要使用 @c expr(var) 将 VariableIndex 强转为 ExpressionHandle。
 *
 * @param expr 非线性目标项表达式句柄 @c ExpressionHandle。
 *             支持sin/cos/tan/exp/log/pow等可微非线性运算，也支持线性、二次表达式。
 *
 * @return void 无返回值。
 *
 *
 * @code
 * // 正确示例：RAII守卫包裹表达式构建与添加非线性目标
 * {
 *     ExpressionGraphContextGuard guard_con1;
 *
 *     // 简单三角函数非线性目标项
 *     model.add_nl_objective( sin( expr(x1) + expr(x2) ) );
 *
 *     // 复合表达式，变量通过 expr() 转换为 ExpressionHandle
 *     model.add_nl_objective( expr(x1) * expr(x4) * (expr(x1)+expr(x2)+expr(x3)) + expr(x3) );
 * }
 * // guard_con1 析构：局部的ExpressionHandle失效；已添加的非线性目标项在模型中保持有效。
 *
 * @endcode
 */
void add_nl_objective(const ExpressionHandle &expr);
```

## 求解

模型求解时使用

```cpp
void optimize();
```

```cpp
/**
 * @brief 执行优化求解，调用Ipopt求解器对当前模型进行求解
 *
 * 将已构建完成的变量、约束、目标函数提交给求解器执行数值优化计算。
 * 求解完成后，变量的求解结果会保存至模型内部，可通过接口读取变量最优值、约束对偶等信息。
 *
 * @return void 无返回值。
 *
 * @attention
 * - 调用 @c optimize() 前模型必须完成必要配置：
 *   1. 已添加至少一个优化变量；
 *   2. 通过 @ref set_objective() 设置目标函数与优化方向；
 *   3. 按需添加线性、二次、非线性约束、非线性目标项。
 * - 求解失败时，模型内部会保留求解器返回状态信息，可通过状态查询接口获取失败原因。
 *
 * @note
 * - 本函数为阻塞调用；求解未完成时不会返回。
 * - 求解结束后，可读取VariableIndex对应的变量解；
 * - 支持对同一个模型实例多次调用 optimize()，每次调用都会重新执行一次求解。
 * - @c ExpressionGraphContextGuard 不影响本接口，求解阶段不再依赖表达式栈RAII上下文。
 *
 * @code
 * // 完整求解流程示例
 * // 1. 添加变量
 * auto x1 = model.add_variable(0.0, 10.0, VariableDomain::Continuous, "x1");
 * auto x2 = model.add_variable(0.0, 10.0, VariableDomain::Continuous, "x2");
 *
 * // 2. 设置目标函数
 * model.set_objective(x1+x2, ObjectiveSense::Minimize);
 *
 * // 3. 添加约束
 * {
 *     ExpressionGraphContextGuard guard;
 *     model.add_nl_constraint(expr(x1)+expr(x2), ConstraintSense::LessEqual, 8.0);
 * }
 *
 * // 4. 执行求解（阻塞调用）
 * model.optimize();
 *
 * // 5. 获取变量求解结果
 * double val = model.get_variable_value(x1);
 * @endcode
 */
void optimize();
```

## 求解器参数设置

设置求解器的参数

```cpp
/**
 * @brief 通过参数名设置Ipopt、Gurobi、HiGHS等求解器选项参数
 *
 * 重载接口，按参数类型分别设置浮点、整数、字符串类型的求解器参数。
 * 参数名称与求解器原生选项名称保持一致，参数会在后续 @ref optimize() 求解时生效。
 *
 * @param name 求解器选项名称，C字符串，与官方选项名完全匹配，大小写敏感。
 * @param value 浮点(double)类型参数值。
 *
 * @return void 无返回值。
 *
 * @attention
 * - 必须在调用 @ref optimize() **之前**调用本系列接口；求解开始后再设置参数不会对本次求解生效。
 * - 参数名严格遵循Ipopt官方选项，名称错误、类型不匹配不会报错。
 * - 多次设置同一个name，后设置的值覆盖之前的值。
 *
 * @note
 * - 三套重载分别对应 double / int / const char*，根据参数实际类型选择对应重载。
 * - 参考官方文档查看全部可用参数名称与取值范围。
 *
 * @code
 * // 设置浮点参数：收敛容差
 * model.set_parameter("tol", 1e‑6);
 *
 * // 设置整数参数：最大迭代次数
 * model.set_parameter("max_iter", 1000);
 *
 * // 设置字符串参数：输出日志级别
 * model.set_parameter("print_level", "5");
 * @endcode
 */
void set_parameter(const char *name, double value);

/**
 * @brief 通过参数名设置求解器选项参数（整数版本）
 *
 * @param name 求解器选项名称，C字符串，与官方选项名完全匹配，大小写敏感。
 * @param value 整数(int)类型参数值。
 *
 * @return void 无返回值。
 *
 * @sa set_parameter(const char *, double)
 */
void set_parameter(const char *name, int value);

/**
 * @brief 通过参数名设置求解器选项参数（字符串版本）
 *
 * @param name 求解器选项名称，C字符串，与官方选项名完全匹配，大小写敏感。
 * @param value 字符串类型参数值；函数内部拷贝字符串内容，外部传入字符串调用后可释放。
 *
 * @return void 无返回值。
 *
 * @sa set_parameter(const char *, double)
 */
void set_parameter(const char *name, const char *value);
```

### 获取变量属性

变量属性通过接口获得：

```cpp
double get_variable_attribute(const VariableIndex& variable, VariableAttributeDouble attribute);
int get_variable_attribute(const VariableIndex& variable, VariableAttributeInt attribute) ;
std::string get_variable_attribute(const VariableIndex& variable, VariableAttributeString attribute) override;
```

```cpp
/**
 * @brief 获取变量浮点类型属性
 *
 * 通用接口，读取指定变量的浮点型属性值。该接口抽象底层求解器。
 * 属性由枚举 VariableAttributeDouble 指定。
 *
 * @param variable 目标变量索引句柄 VariableIndex，必须属于当前模型实例。
 * @param attribute 浮点属性枚举 VariableAttributeDouble，指定要读取的属性项，例如变量下界、上界、求解后最优值等。
 *
 * @return double 返回对应属性的浮点数值。
 *
 * @attention
 * - 部分属性（如求解最优值）需要在 @ref optimize() 执行完成之后才有有效结果；求解前读取可能返回无效值。
 *
 * @sa VariableAttributeDouble
 * @sa get_variable_attribute(const VariableIndex&, VariableAttributeInt)
 * @sa get_variable_attribute(const VariableIndex&, VariableAttributeString)
 *
 * @code
 * // 获取变量求解后的最优值
 * double x_opt = model.get_variable_attribute(x, VariableAttributeDouble::PrimalValue);
 * // 获取变量下界
 * double lb = model.get_variable_attribute(x, VariableAttributeDouble::LowerBound);
 * @endcode
 */
double get_variable_attribute(const VariableIndex& variable, VariableAttributeDouble attribute);

/**
 * @brief 获取变量整数类型属性
 *
 * 通用接口，读取指定变量的整数型属性值。该接口抽象底层求解器，不限于Ipopt。
 * 属性由枚举 VariableAttributeInt 指定。
 *
 * @param variable 目标变量索引句柄 VariableIndex，必须属于当前模型实例。
 * @param attribute 整数属性枚举 VariableAttributeInt，指定要读取的属性项，例如变量域类型标识、求解状态标记等。
 *
 * @return int 返回对应属性的整数值。
 *
 * @attention
 * - 部分属性需要在 @ref optimize() 求解完成后才可获得有效数据。
 * - 属性支持情况取决于后端求解器。
 *
 * @sa VariableAttributeInt
 */
int get_variable_attribute(const VariableIndex& variable, VariableAttributeInt attribute) ;

/**
 * @brief 获取变量字符串类型属性
 *
 * 通用接口，读取指定变量的字符串类型属性，例如变量名称。该接口抽象底层求解器。
 * 属性由枚举 VariableAttributeString 指定。
 *
 * @param variable 目标变量索引句柄 VariableIndex，必须属于当前模型实例。
 * @param attribute 字符串属性枚举 VariableAttributeString，指定要读取的属性项，如变量名称。
 *
 * @return std::string 返回对应属性字符串；无名称时返回空字符串。
 *
 * @attention
 * - 属性支持情况取决于后端求解器。
 *
 * @sa VariableAttributeString
 *
 * @code
 * std::string var_name = model.get_variable_attribute(x, VariableAttributeString::Name);
 * @endcode
 */
std::string get_variable_attribute(const VariableIndex& variable, VariableAttributeString attribute);
```



## 属性

### 约束比较属性

```cpp
/**
 * @brief 约束比较方向枚举
 *
 * 用于线性、二次约束，定义约束左侧表达式与右侧 rhs 的比较关系。
 */
enum class ConstraintSense
{
    /// \f$ f \le rhs \f$ 小于等于
    LessEqual,
    /// \f$ f \ge rhs \f$ 大于等于
    GreaterEqual,
    /// \f$ f = rhs \f$ 等于
    Equal,
};
```

### 目标优化方向

```cpp
/**
 * @brief 目标函数优化方向枚举
 */
enum class ObjectiveSense
{
    /// 最小化目标函数
    Minimize,
    /// 最大化目标函数
    Maximize
};
```

### 变量属性

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

