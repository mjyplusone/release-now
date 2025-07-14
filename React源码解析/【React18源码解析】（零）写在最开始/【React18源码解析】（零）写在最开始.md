作为一个前端程序猿，React 是我工作中的重要工具。搞清楚它的源码实现，才能在使用它时**知其然，知其所以然**。所以，我花了一些时间阅读它的源码，本系列 **React18 源码解析**，算是对自己学习过程的一个记录和分享吧。

# 版本迭代

我们知道，React 从 2013 年发布以来，经历了多个关键版本的更新，整体架构也发生了较大的变化。

### React 0.x - 15.x

早期阶段，引入 JSX 写法，虚拟 DOM，基本的 class 组件。

### React 16

首次引入 Fiber 架构，为后续并发渲染打下基础。

### React 16.8

hooks 正式发布，函数式组件具备状态、副作用等功能，鼓励使用函数式组件代替 class 组件。

### React 18

- 并发渲染（Concurrent Rendering）正式上线。
- `startTransition` 标记低优先级更新。
- 自动批处理等。

### React 19

- `useFormStatus` / `useFormState` 管理表单提交、错误、状态。
- `useOptimistic` 支持乐观更新（先展示、后提交）。
- 准备迎接 React Compiler（自动性能优化）。

这些更新中，React18 是一个具有重大意义的版本。既 React16 引入 Fiber 后，又一次大的架构升级，它改变渲染模型，开启了并发渲染的时代。所以，本系列中，我们将针对 **React 18** 的源码进行深入解析。

# 获取源码

首先，从 github 找到 React 的[源码](https://github.com/facebook/react)，`git clone` 到本地。

然后查看 18.x 系列的 tag，并 `git checkout` 到相应版本。

```
git tag | grep v18

git checkout v18.0.0
```

# 目录结构

React 源码使用 monorepo 的形式管理。

## 根目录

根目录结构大致如下：

```
nestjs/
├── packages/
├── scripts/
├── fixtures/
├── ...
└── package.json
```

其中每个子目录的分工为：

- `packages`：核心源码模块。
- `scripts`：各种脚本，构建发布、性能基准测试等。
- `fixtures`：用于测试的示例项目（如 dom、concurrent 模式等）。

## packages 目录

下面来看下包含核心源码的 `packages` 目录：

```
packages/
├── react/
├── react-dom/
├── react-reconciler/
├── scheduler/
├── shared/
├── react-server/
├── react-native-renderer/
├── react-devtools/
└── ...
```

### react

React 对外暴露的顶层 API。比如 `React.createElement` 、hooks API (`useState`、`useEffect`) 等。

### react-dom

处理 DOM 渲染逻辑，与浏览器 DOM 操作直接相关的 API。比如 `ReactDOM.render` 、`ReactDOM.creteRoot` 、`ReactDOM.createPortal` 等。

### react-reconciler

核心调和器，包含 Fiber 树构建、调和（diff）逻辑、更新等。

### scheduler

调度器，用于控制任务优先级、任务延迟、中断恢复等，实现并发渲染。

### shared

多个包之间共享的通用工具函数和常量。

### react-server

用于服务端组件（React Server Components）。

### react-native-renderer

React Native 渲染器。

### react-devtools

React 开发者工具。

# 总结

本章介绍了 React 版本的迭代过程以及源码的目录结构。源码按照功能模块，通过 monorepo 的形式组织，其中比较核心的是 react、react-dom、react-reconciler、scheduler 这几个包，后续我们会围绕它们的实现深入解析。
