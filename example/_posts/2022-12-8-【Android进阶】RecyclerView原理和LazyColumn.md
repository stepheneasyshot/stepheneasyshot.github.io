---
layout: post
description: > 
  本文介绍了RecyclerView的优化原理，和Compose中的LazyColumn组件的实现原理。
image: 
  path: /assets/img/blog/blogs_recyclerview_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_recyclerview_cover.png
    960w:  /assets/img/blog/blogs_recyclerview_cover.png
    480w:  /assets/img/blog/blogs_recyclerview_cover.png
accent_image: /assets/img/blog/blogs_recyclerview_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android进阶】RecyclerView原理和LazyColumn
## RecyclerView的优化点
最初，要在Android界面中显示一个列表，使用的组件是 `ListView` ，但是由于 `ListView` 的性能问题，在Android 5.0之后，Google引入了 `RecyclerView` 组件。 `RecyclerView` 提供一个高度可定制的列表视图，同时保持了良好的性能和用户体验。

`RecyclerView` 用于在有限的屏幕空间内显示大量数据列表或网格。它是 `ListView` 和 `GridView` 的升级版，提供了更好的性能和灵活性。

**核心理念：视图回收 (View Recycling)**

`RecyclerView` 的核心优化点是 **视图回收机制** 。当列表中的项滚动出屏幕时，`RecyclerView` 不会销毁其视图。相反，它会**回收**这些不再可见的视图，并将其重新用于屏幕上即将显示的新项。通过视图回收机制，显著减少了视图创建的开销，尤其是在处理大量数据时表现出色。

**对比ListView** 来说，ListView 的视图回收机制依赖于开发者在 `getView()` 方法中手动实现 `ViewHolder` 模式来缓存子视图的引用 (`findViewById()` 操作耗时)。如果开发者不使用 `ViewHolder` 模式，那么每次 `getView()` 被调用时（即使是重用 convertView），都会重复调用 `findViewById()` 来查找子视图，这会严重影响滚动性能。

> `findViewById()` 的实现方式是从当前视图（通常是 Activity 的根视图或一个 ViewGroup）开始，递归地遍历整个视图树，查找具有指定 ID 的视图。具体的遍历算法可能是深度优先搜索（DFS）或广度优先搜索（BFS），但无论哪种，它都必须检查视图树中的每一个节点，直到找到匹配的 ID 或者遍历完整个树。

RecyclerView 则强制并内置了 `ViewHolder` 模式，要求你必须创建并使用 ViewHolder 来持有视图引用，从而从根本上解决了这个问题。

**动画**

动画实现难度方面，ListView 在添加、删除或移动 item 时，实现动画效果非常复杂，通常需要手动处理和控制。RecyclerView 内置了 ItemAnimator 接口，使得为列表项的增删改提供平滑的动画变得非常简单和优雅。

**职责解耦**

ListView 将视图回收、布局管理和数据绑定等职责都集中在 ListView 和 Adapter 内部，导致代码耦合度较高。
RecyclerView 将这些职责分离到独立的组件中 (Adapter、ViewHolder、LayoutManager、ItemAnimator、ItemDecoration)，使得组件更加解耦，易于测试、维护和扩展。

**数据更新效率**

ListView 只有一个 notifyDataSetChanged() 方法来通知数据变化。这意味着即使只有一个 item 发生了变化，整个列表也可能需要重新绘制，效率低下。

RecyclerView 提供了更精细的数据更新通知方法，如 notifyItemInserted()、notifyItemRemoved()、notifyItemChanged() 等，这些方法可以告知 RecyclerView 具体哪些 item 发生了变化，从而实现局部更新和更流畅的动画。

### `RecyclerView` 的主要组成部分

要使用 `RecyclerView`，通常需要以下几个关键组件协同工作：

1.  **`RecyclerView` 本身 (The `ViewGroup`)**:
    * `RecyclerView` 是一个 `ViewGroup`，它负责承载和管理列表中的所有视图。
    * 你把它添加到你的布局文件中，就像添加任何其他 UI 元素一样。

2.  **`ViewHolder` (视图持有者)**:
    * 列表中的每个独立元素都由一个 `ViewHolder` 对象进行定义。
    * `ViewHolder` 的作用是持有并提供对单个列表项布局中所有视图的引用（例如 `TextView`、`ImageView` 等）。
    * 当你创建 `ViewHolder` 时，它还没有任何关联的数据。`RecyclerView` 会在需要时将其绑定到其数据。
    * 你需要通过扩展 `RecyclerView.ViewHolder` 来定义自己的 `ViewHolder` 类。

3.  **`Adapter` (适配器)**:
    * `Adapter` 负责将你的数据与 `ViewHolder` 绑定，并管理列表项的创建和更新。
    * `RecyclerView` 通过在 `Adapter` 中调用方法来请求视图并将视图绑定到其数据。
    * 你需要通过扩展 `RecyclerView.Adapter` 来定义自己的 `Adapter` 类。
    * `Adapter` 主要有三个重要方法：
        * `onCreateViewHolder()`: 当 `RecyclerView` 需要一个新的 `ViewHolder` 来表示列表项时，会调用此方法。你在这里创建 `ViewHolder` 及其关联的视图布局。
        * `onBindViewHolder()`: 当 `RecyclerView` 准备好将数据绑定到 `ViewHolder` 时，会调用此方法。你在这里获取特定位置的数据，并将其填充到 `ViewHolder` 的视图中。
        * `getItemCount()`: 返回列表中项的总数。

4.  **`LayoutManager` (布局管理器)**:
    * `LayoutManager` 负责在 `RecyclerView` 中定位和排列列表中的各个元素，并决定何时回收和重用不再可见的项视图。
    * `RecyclerView` 库提供了几种开箱即用的 `LayoutManager`：
        * `LinearLayoutManager`: 将项排列成一维列表（垂直或水平滚动）。
        * `GridLayoutManager`: 将项排列成二维网格。
        * `StaggeredGridLayoutManager`: 将项排列成错列的二维网格，每列稍微偏移。
    * 如果这些内置的 `LayoutManager` 不符合你的需求，你也可以通过扩展 `RecyclerView.LayoutManager` 抽象类来创建自定义的布局管理器。

### 视图回收的详细流程
`RecyclerView` 在处理每个子项视图时，采用了一套高度优化和解耦的机制，旨在实现高性能的列表滚动，尤其是在处理大量数据时。核心是**视图回收 (View Recycling)** 和**职责分离 (Separation of Concerns)**。

下面详细介绍 `RecyclerView` 是如何处理每个子项视图的：

### 1. `LayoutManager`：布局与可见性管理

`LayoutManager` 是 `RecyclerView` 处理子项视图的第一个关键参与者。它的主要职责包括：

* **布局 (Layout)**：决定列表项在 `RecyclerView` 中的排列方式，例如垂直线性、水平线性、网格或瀑布流等。它负责测量和放置每个可见的子视图。
* **滚动 (Scrolling)**：管理用户的滚动事件，并根据滚动方向和速度决定哪些视图应该进入屏幕，哪些应该离开屏幕。
* **视图附着/分离 (Attach/Detach Views)**：当视图进入屏幕时，`LayoutManager` 会将其**附着 (attach)** 到 `RecyclerView`；当视图离开屏幕时，它会将其**分离 (detach)**。这里的分离并不是销毁，而是将其从 `RecyclerView` 的视图层级中移除，但保留在缓存中。
* **视图回收/重用策略 (Recycling/Reusing Strategy)**：`LayoutManager` 会与 `Recycler` 合作，决定何时回收视图（当视图离开屏幕）以及何时重用视图（当需要显示新项时）。

当用户滚动 `RecyclerView` 时，`LayoutManager` 会不断计算哪些数据项应该可见。对于这些可见的数据项：

* 如果有一个可以重用的**废弃 (scrap)** 或**回收 (recycled)** 视图可用，`LayoutManager` 会尝试使用它。
* 如果没有可重用的视图，`LayoutManager` 会通知 `Adapter` 创建一个新的视图。

### 2. `Recycler` (缓存机制)：视图回收池

`RecyclerView` 内部有一个强大的 `Recycler` 机制，它维护了 **多个视图缓存池** ，以高效地管理视图的回收和重用：

* **Scrap Heap (废弃堆)**:
    * 这是一个非常轻量的缓存，用于存储最近被分离但可能很快再次附着的视图。
    * 例如，当你执行一个微小的滚动，或者进行一个 `notifyItemChanged()` 操作时，视图可能只是暂时离开屏幕，然后又回来。
    * 这里的视图不会被解除绑定 (`unbound`)，因此不需要再次调用 `onBindViewHolder()`。
    * `LayoutManager` 会优先从这里查找可重用的视图。

* **View Cache (视图缓存)**:
    * 这个缓存池存储的是最近滚出屏幕的 `ViewHolder`，它们已经被从 `RecyclerView` 中分离。
    * 当一个 `ViewHolder` 从 `Scrap Heap` 无法被重用时，`LayoutManager` 会尝试从 `View Cache` 中获取。
    * `View Cache` 中的 `ViewHolder` 仍然持有视图引用，但它们可能已经与之前的数据解绑，需要通过 `onBindViewHolder()` 重新绑定新数据。
    * 默认情况下，这个缓存的大小是有限的（通常为 2）。

* **RecycledViewPool (回收视图池)**:
    * 这是一个更深层的缓存，存储的是已经完全回收的 `ViewHolder`。
    * 当 `ViewHolder` 离开 `View Cache` 或 `LayoutManager` 明确将其回收时，它会进入 `RecycledViewPool`。
    * 这里的 `ViewHolder` 是按视图类型 (view type) 进行分类存储的。如果你的 `RecyclerView` 有多种不同的 `item` 布局，它们会分别存储在各自的池中。
    * 从 `RecycledViewPool` 取出的 `ViewHolder` **必须**重新绑定数据，即总是会调用 `onBindViewHolder()`。
    * 这个池是可以在多个 `RecyclerView` 实例之间共享的（例如在嵌套 `RecyclerView` 中），进一步提高了效率。

* **Attached Views (已附着视图)**:
    * 这些是当前屏幕上可见的、已经被附着到 `RecyclerView` 中的视图。它们没有被回收，也没有进入任何缓存池。

### 3. `Adapter`：数据与视图的桥梁

`Adapter` 是数据和视图之间的桥梁，它与 `LayoutManager` 和 `ViewHolder` 紧密协作，负责以下工作：

* **`getItemCount()`**: 告诉 `RecyclerView` 总共有多少个数据项。
* **`getItemViewType(int position)`**:
    * 如果你的列表有不同类型的 `item` 布局（例如，一个列表项是图片，另一个是文字），你需要重写这个方法，返回一个唯一的整数来标识不同类型。
    * `RecyclerView` 会根据 `viewType` 从 `RecycledViewPool` 中查找相应类型的 `ViewHolder` 进行重用，避免混淆不同布局的视图。
* **`onCreateViewHolder(ViewGroup parent, int viewType)`**:
    * 当 `LayoutManager` 需要一个新的 `ViewHolder` 时（即 `Scrap Heap` 和 `View Cache` 都没有可重用的视图，或者需要一个新类型的视图时），会调用此方法。
    * 在这个方法中，你通过 `LayoutInflater.from(parent.getContext()).inflate()` **创建**一个新的视图布局。
    * 然后，你将这个视图传递给你的自定义 `ViewHolder` 构造函数，`ViewHolder` 会在这里通过 `findViewById()` 查找并缓存其内部的子视图引用。
    * 最后，返回这个新创建的 `ViewHolder` 实例。**这个方法通常只会被调用有限的次数**，因为一旦创建了足够多的视图来填充屏幕，就会开始进行视图回收。
* **`onBindViewHolder(ViewHolder holder, int position)`**:
    * 当 `LayoutManager` 需要将一个 `ViewHolder` 与特定位置的数据项关联起来时，会调用此方法。
    * 无论 `ViewHolder` 是新创建的还是从缓存中重用的，这个方法都会被调用。
    * 你在这里从数据源中获取 `position` 对应的数据，然后使用 `holder` 中缓存的子视图引用，将数据填充到视图中（例如 `holder.textView.setText(data.getName())`）。这是数据绑定的核心步骤。

### 4. `ViewHolder`：视图引用持有者

`ViewHolder` 是 `RecyclerView` 性能优化的核心。它的作用是：

* **缓存视图引用**: 在 `ViewHolder` 的构造函数中，通过 `findViewById()` 获取所有需要操作的子视图的引用，并将其存储为成员变量。
* **避免重复查找**: 一旦 `ViewHolder` 被创建并缓存了视图引用，后续无论这个 `ViewHolder` 被重用多少次，都无需再次调用 `findViewById()`。直接通过 `ViewHolder` 内部的成员变量即可访问子视图，大大提高了性能。
* **封装逻辑**: `ViewHolder` 也可以作为放置与单个列表项相关的事件监听器（如点击事件）和特定UI更新逻辑的好地方。

### 总结整个流程：

1.  **初始化**: `RecyclerView` 被添加到布局中。
2.  **设置 `LayoutManager`**: `RecyclerView` 知道如何排列其子项。
3.  **设置 `Adapter`**: `RecyclerView` 知道如何获取数据并创建/绑定视图。
4.  **初始布局**: `LayoutManager` 向 `Adapter` 请求足够多的 `ViewHolder` (`onCreateViewHolder`) 并绑定数据 (`onBindViewHolder`) 来填充屏幕，然后将这些 `ViewHolder` 的视图附着到 `RecyclerView`。
5.  **滚动时**:
    * 当一个 `item` 滚出屏幕时，`LayoutManager` 会将其视图从 `RecyclerView` 中**分离**。这个 `ViewHolder` 可能会进入 `Scrap Heap` 或 `View Cache`，最终可能进入 `RecycledViewPool`。
    * 当一个新 `item` 需要进入屏幕时，`LayoutManager` 首先尝试从 `Scrap Heap` 中获取一个可重用的 `ViewHolder`。
    * 如果 `Scrap Heap` 中没有，它会尝试从 `View Cache` 中获取。
    * 如果 `View Cache` 中也没有，它会检查 `RecycledViewPool` 中是否有指定 `viewType` 的 `ViewHolder`。
    * 如果所有缓存中都没有，`LayoutManager` 会通知 `Adapter` 调用 `onCreateViewHolder()` 来创建一个全新的 `ViewHolder`。
    * 一旦获取到 `ViewHolder`（无论是重用的还是新的），`LayoutManager` 会通知 `Adapter` 调用 `onBindViewHolder()`，将当前位置的数据绑定到 `ViewHolder` 的视图上。
    * 最后，`LayoutManager` 将这个绑定好数据的 `ViewHolder` 的视图**附着**到 `RecyclerView` 中，使其可见。

通过这种精巧的视图回收和职责分离机制，`RecyclerView` 能够以极高的效率处理动态列表，无论是数量庞大的数据还是复杂的 `item` 布局，都能提供流畅的用户体验。

## LazyColumn
在 Jetpack Compose 中，`LazyColumn` (以及 `LazyRow`、`LazyVerticalGrid` 等 `Lazy` 布局) 是处理大量列表数据的核心组件。与 Android View 系统中的 `RecyclerView` 类似，`LazyColumn` 的显示逻辑也基于**按需组合 (Composition on Demand)** 和**视图回收 (View Recycling)** 的概念，但它的实现方式与 `RecyclerView` 略有不同，并且更加“Compose 式”。

`LazyColumn` 是一个 **懒加载 (Lazy Loading)** 的列表，它只会在列表项进入屏幕可见区域时才会创建和渲染这些项。这意味着，当列表中有大量数据时，它可以显著减少内存占用和渲染性能。

### `LazyColumn` 的核心显示逻辑

`LazyColumn` 的设计目标是：**只组合（Compose）并测量（Measure）当前在屏幕上可见或即将可见的 `item`，而不是一次性处理所有数据项。**

1.  **按需组合 (Composition on Demand)**:
    * 当你向 `LazyColumn` 提供一个数据列表时，它并不会立即为列表中的所有数据项创建对应的 Composable 函数实例。
    * 相反，它只会根据当前滚动位置和屏幕尺寸，计算出哪些 `item` 应该显示在屏幕上。
    * **只有那些位于可见区域内的 `item` 的 Composable 函数才会被执行 (Compose)**。这被称为“按需组合”。
    * 当你滚动列表时，新的 `item` 进入可见区域，它们的 Composable 函数才会被调用，从而创建其 UI。滚出屏幕的 `item` 的 Composable 函数会停止执行，其对应的 UI 节点也会被销毁。

2.  **内容插槽 (Content Slotting)**:
    * `LazyColumn` 使用了 Compose 的“内容插槽”模式。你不是直接将 Composable 函数传递给 `LazyColumn`，而是通过 `items` 或 `item` DSL（领域特定语言）块来定义每个 `item` 的内容。
    * `LazyColumn { items(myList) { data -> MyItemComposable(data) } }`
    * 这里的 `MyItemComposable(data)` 就是一个内容插槽，`LazyColumn` 会根据需要来组合这些内容。

3.  **智能回收与重用 (Smart Recycling and Recomposition)**:
    * 虽然 Compose 不像 `RecyclerView` 那样有显式的 `ViewHolder` 概念，但 `LazyColumn` 内部也实现了高效的回收机制。
    * 在 `LazyColumn` 组件中，能够被“复用”的主要概念是 Composable 函数的 UI 结构和底层布局对象 (layout objects)，而不是像传统 RecyclerView 那样对 View 实例进行回收和重用。
    * 当另一个 `item` 需要进入屏幕时，如果缓存中存在一个相同**类型**的 Composable 实例，并且这个实例可以被重用，`LazyColumn` 会尝试重用它。
    * **重用的核心在于 Recomposition (重组)**：
        * 如果被重用的 Composable 实例接收到的数据（或状态）与上次相同，那么 Composable 可能会跳过执行（可跳过 Composable 的优化）。
        * 如果数据不同，Compose 会进行**智能重组**。它会比较新的数据和旧的数据，并只更新 UI 中实际发生变化的部分，而不是重新创建整个 `item` 的 UI。这种增量更新是 Compose 性能的关键。

4.  **`LazyListState` 和滚动位置管理**:
    * `LazyColumn` 内部维护着一个 `LazyListState` 对象（通常通过 `rememberLazyListState()` 创建）。
    * `LazyListState` 记录了当前列表的滚动位置、第一个可见 `item` 的索引、可见 `item` 的偏移量等信息。
    * 当用户滚动时，`LazyListState` 会更新，并通知 `LazyColumn` 重新测量和布局可见 `item`。

5.  **`key` 参数的重要性**:
    * `items` 和 `item` DSL 提供了 `key` 参数，强烈建议为每个 `item` 提供一个稳定且唯一的 `key`。
    * **`LazyColumn` 使用 `key` 来优化重组和识别 `item` 的移动、添加或删除。**
    * 如果不提供 `key`，`LazyColumn` 默认使用 `item` 的索引作为 `key`。当列表中的 `item` 顺序发生变化（例如删除或重新排序）时，使用索引作为 `key` 会导致错误的重用或不必要的重组，甚至可能导致动画效果不佳。
    * 提供了稳定的 `key` 后，即使数据列表的顺序发生变化，`LazyColumn` 也能识别出哪些是同一个逻辑 `item`，从而正确地重用其 Composable 实例并应用正确的动画。
    * 需要注意的是每个item的key应该唯一，如果运行时出现重复的key时会直接报错崩溃。

### `LazyColumn` 与 `RecyclerView` 的主要区别 (在视图处理层面)

| 特性             | `RecyclerView` (View System)                                   | `LazyColumn` (Jetpack Compose)                               |
| :--------------- | :------------------------------------------------------------- | :----------------------------------------------------------- |
| **视图创建** | 通过 `Adapter` 的 `onCreateViewHolder` 方法，使用 XML inflated 创建 `View` 实例。 | 通过 Composable 函数的执行（Composition），直接生成 UI 节点。   |
| **视图回收** | `ViewHolder` 模式，通过 `Recycler` 缓存 `View` 对象。             | 内部缓存 Composable 实例，通过 Recomposition 机制重用和更新 UI。没有显式的 `ViewHolder` 类。 |
| **数据绑定** | `Adapter` 的 `onBindViewHolder` 方法负责将数据绑定到 `View` 的各个子组件。 | 数据直接作为参数传递给 `item` 的 Composable 函数，Compose 自动处理数据变化时的 Recomposition。 |
| **布局管理** | 通过独立的 `LayoutManager` 类 (`LinearLayoutManager` 等) 管理布局策略。 | `LazyColumn` 自身内置了布局逻辑，无需单独的 `LayoutManager` 类。 |
| **动画** | 通过 `ItemAnimator` 实现增删改动画，通常需要额外配置。           | 内置更平滑的动画支持，得益于 Compose 的 Recomposition 和 Keying 机制。 |
| **强制性优化** | `ViewHolder` 模式需要开发者手动实现才能获得最佳性能。            | `LazyColumn` 内部已经强制并内置了类似的优化，开发者只需关注 `item` 的 Composable 逻辑。 |
| **UI 范式** | 命令式 UI：手动操作 `View` 对象。                                | 声明式 UI：描述 UI 应该是什么样子，Compose 负责如何达到。       |

### 总结

`LazyColumn` 的显示逻辑是基于 Compose 的声明式 UI 特性，它在运行时智能地组合和重组 `item` 的 Composable 函数，只渲染当前可见的 UI 部分。通过内部的缓存机制和 `key` 参数的优化，`LazyColumn` 实现了类似 `RecyclerView` 的高效列表性能，同时提供了更加简洁和直观的 API。开发者不再需要关心繁琐的 `findViewById`、`ViewHolder` 和 `Adapter` 生命周期管理，只需专注于定义每个 `item` 的 UI 外观和数据绑定逻辑。
