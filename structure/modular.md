# 模块化

本系统中包含各种具体技术实现内容，为了规范这些技术的具体实现，我们称系统中所有使用到的技术为**模块**，并且它们都继承与一个统一的接口 `BasicModule`，通过 `OOP` 中继承和多态的思想，不仅可以规范和约束模块实现，也能提供模块特定的接口。

## 属性

其中 **可见性** 主要用于描述属性内容是否能被外界访问。

| 名称         | 字段     | 默认值   | 可见性 | 备注                                          |
| ------------ | -------- | -------- | ------ | --------------------------------------------- |
| 名称         | name     | 必须     | yes    |                                               |
| 别名         | alias    | 必须     | yes    | 模块对应的中文名                              |
| 实现类型     | kind     | `basic`  | yes    | `basic` 指默认（基础）实现, `null` 指为空实现 |
| 实现类型列表 | kinds    | `[]`     | yes    | 存储目前已经支持的实现类型                    |
| 非空性       | not_null | `true`   | no     | 记录该模块是否能被加载为空                    |
| 路径         | path     | ` `      | no     | 模块代码所在路径, `module` 目录的相对路径     |
| 父模块       | sup      | 程序生成 | yes    | 父模块名                                      |
| 子模块列表   | sub      | `[]`     | yes    | 包含的子模块名                                |
| 嵌套深度     | depth    | 程序生成 | no     | 记录子模块的嵌套深度                          |

<!-- 由于模块状态属于模块运行阶段的属性，不属于模块的持久化信息，因此保存在模块管理单元内。 -->

> **注意**：模块中仅记录静态的模块信息，有些属性会随着运行的状态动态切换，此部分属性会考虑在后续迁移到 `ModuleManageCell` 中。

### 1. 实现类型

每个模块可以存在多个具体的实现类型，
但是本系统保留了 2 个实现类型名称 `basic` 和 `null`。

-   `basic`：模块的的基础实现类型，对于一些**组织者模块**，它一般来说只有一种实现类型，那么就可以使用 `basic` 称呼。除此之外，如果你不知道这个模块实现需要起什么名字，那你就可以使用 `basic` 了。在大多数情况下，其并没有提供任何额外的语义信息，但是在系统配置和动态导入时，其会提供不少的便利性。
-   `null`：空实现，其在动态导入过程时会直接返回 `None`。模块是否能设置未空，也由其**非空性**决定。引入空实现的主要原因是用于关闭一些非核心的模块。当然，在进行父模块功能开发时也需要注意这些空模块的调用问题。

### 2. 模块嵌套

在本系统中，模块是允许被嵌套的，一个模块允许拥有多个子模块。也就是说，本系统也是一个巨大的模块。

模块嵌套的必然结果是构建了一颗模块依赖树（不考虑一个模块会被多个模块依赖的情况），那么对于叶节点模块模块来说，他们必须需要提供功能，除非它的实现类型为 `null`，否则不实现具体功能的模块是无意义的，所以我们称之为**功能性模块**（`functional module`）。而非叶子节点则属于组织者模块（`organizer module`），它是通过依靠子模块来实现复杂的逻辑处理。比如说在本系统中，`bot` 需要使用 `searcher` 检索专业知识数据，并结合提示词供词，最终通过 `caller` 调用大语言模型，以生成问题对应的答案。

模块嵌套为系统开发提供了两个好处，首先是其实现了逻辑和实现解耦，子模块只需要关心具体的技术实现细节，父模块则关注如何组织这些子模块的逻辑即可。其次，模块嵌套也建立了一套父子模块的依赖链，这为后续模块的级联控制打下了结构基础。

### 3. 可控模块

本系统提供更细粒度控制模块逻辑，
我们称 `booter` 的一级子模块为**可控模块**（当然其本身也是）。
一方面为了更细粒度地控制不同模块的启停，
避免系统切换开销过大。
另一方面，则是并不是所有模块都需要控制，
而且单独启动也并没有意义（除了程序调试）。

当然，系统通过引入`null`实现类型，来补充对模块的控制能力。
当模块实现被设置为`null`时，系统在启动时就不会加载该模块，
自然不会启动该模块。
一个模块都否被设置为空实现由`not_null`属性决定。

> 目前仅在 Webui 控制器对其进行限制。

## 生命周期

模块的生命周期包括两个阶段，模块的加载阶段和运行阶段。前者负责动态导入对应代码中的类，并进行实例化的过程，而后者则是将覆盖模块停止和运行之间的来回切换的过程。

值得注意的是，由于控制反转（详情请见 [模块管理器](./manager.md) 部分）的机制，因此对于模块生命周期的控制权并不在模块本身，而是在模块管理器上，但实际的逻辑仍需要模块中提供的信息和代码逻辑进行处理和控制。

### 1. 加载阶段

加载阶段是导入具体模块实现类型的过程，其主要根据模块属性进行吗拼接组合得到具体的模块代码路径。通过这样约定的模块代码存放逻辑，系统就可以方便和简洁地导入具体模块的实现类型，减少了代码和实际模块之间的耦合度。

导入过程分为模块路径和类名称，主要使用的是以下 3 个模块属性。模块路径为为 `path.name.kind` 其中 `path` 是以项目文件夹 `module` 为基础的相对路径。类的名称使用大驼峰命名法，格式为`KindName`。值得注意的是，当 `kind` 为 `basic` 时，`kind` 就不需要显式地标明，为 `null` 时，将不会进行加载。

-   `path`: 模块所在代码路径
-   `name`: 模块的名字
-   `kind`: 模块的实现类型

以上的模块导入逻辑可以转换为如下代码。

```python
if kind == NULL:
    return None
else kind == BASIC:
    from module.path.name import BasicName
    return BasicName
else:
    from module.path.name.kind import KindName
    return KindName
```

> **注意:** 大驼峰命名法意味着无论配置文件中 `kind` 为什么，
> 最终只会将开头字母大写。

### 2. 运行阶段

模块在运行阶段实际上是模块停止和运行之间的切换，其流程图如下所示，主要分为 4 中状态。

1. 未加载状态：当模块实现类型为 `null` 或加载失败时，则会处于这个阶段
2. 主要状态：相对固定的两个状态，表示模块正常运行或停止，`Stopped` 和 `Started`
3. 中间状态：表示运行和停止之间切换的过程
4. 异常状态：当模块状态在切换过程中，发生异常时会产生的状态

> TODO: 运行阶段的状态运行仍然在优化和完善中...

![生命周期](./img/life_cycle.svg)

#### a. 钩子函数

由于控制反转机制，模块的控制权实际上在管理器中。而模块的实际的启动逻辑则是封装在模块的钩子函数中，管理器在接收到对应的控制指令时，会调用模块中的钩子函数，以执行模块中自定义的处理逻辑。
目前提供的钩子函数如下。

-   “启动子模块”前: `before_starting_submodules`
-   自定义启动逻辑: `handing_starting`
-   自定义停止逻辑: `handing_stopping`
-   “停止子模块”后: `after_stopping_submodules`

#### b. 多线程

某些模块在启动之后，需要持续的运行处理，
因此需要使用多线程的技术。
本系统为了对多线程的创建和回收进行统一的管理，
因此封装了 `make_thread` 用于创建和运行线程。
该函数创建的线程对象会存储在模块内部，
开发者需要自行选择合适的钩子函数作为时机，创建和运行线程。

在线程内部，通常使用死循环的方式进行循环操作，以保证可持续运行。
但是在实际实现上，需要通过标志位来控制线程终止，
以进行优雅地停机。
目前本系统提供 `is_running` 属性，
当模块处于 `Started` 和 `Starting` 时，
该属性为 `true`。
