+++
title = "我的 AI 辅助前端 TDD 工作流"
date = 2026-04-07
draft = false
tags = ["TDD", "AI", "Frontend", "Testing", "MSW"]
+++

## 背景

AI 写代码确实快，但快归快，它有两个比较明显的风险。第一个，改动范围不可控——你让它改一个模块，它有时候会顺手把旁边的东西也改了，连锁反应，别的地方就出问题了。第二个，实现结果不一定对——代码能跑不代表逻辑是对的，AI 有时候会很自信地写出一段看着没问题但其实是错的代码。

<!--more-->

这些问题如果开发阶段没拦住，一路带到提测，那时候再回来排查，成本就远比开发阶段高了。所以我们想的是：**有没有办法在编码阶段就自动把这些问题拦住，而不是靠人去一个一个验？**

我们的答案是 TDD。我在一个内部工具平台项目 Skills Hub 中实践了一套 AI + TDD 的工作流，这篇文章记录核心思路、技术选型和真实案例。

## TDD 工作流：Red → Green → Refactor

整体流程是 Red-Green-Refactor 三步循环，由 AI 辅助驱动。

**第一步，Red 阶段，写测试。** 拿到需求文档和接口文档之后，先让 AI 想清楚：这个功能要验证什么？然后生成对应的测试用例。这时候测试一定是失败的，因为功能代码还没写。这个红是预期内的，它恰好说明测试本身是有效的，不是一跑就绿的摆设。

**第二步，Green 阶段，实现代码。** 核心原则是写最少的代码让测试通过，不过度设计。测试就是验收标准，通过了才算行。这一步可以交给 AI——测试就是它的约束条件，不需要额外的自然语言描述。

**第三步，Refactor 阶段，验证。** 我们不跑全量测试，只跑 git 变更相关的，一条命令 `pnpm test:changed`。它基于 git diff 加模块依赖图，自动算出哪些测试受影响，只跑那些，反馈很快。全绿就进入下一个循环。

关键在于：在 AI 辅助开发的场景下，**测试不再只是质量保障工具，而是你交给 AI 的验收标准**。你定义"什么是对的"，AI 负责"怎么做到"。

## 两个关键技术选择

测试用例分两类，对应两个技术选型。

### data-testid：结构断言的稳定锚点

一类是结构和交互的断言，我们优先用 `data-testid` 来定位元素。为什么不用文本？因为文字、图标、翻译这些在开发过程中经常变，`data-testid` 是稳定的锚点，不会因为 UI 改了一个字测试就挂了。

```tsx
// 用文本断言：切换语言就挂
expect(screen.getByText('开发工具')).toBeInTheDocument()

// 用 data-testid：不管语言怎么切，结构不变就不挂
expect(screen.getByTestId('category-tab-dev')).toBeInTheDocument()
```

`data-testid` 就是一个普通的 HTML attribute，对性能零影响。可以用 babel 插件在生产构建时剥离，但我们选择保留——QA 团队可以复用同一套 testid 做 E2E 测试，不需要维护两套定位策略。

另外一个收益：纯 UI 重构（比如把一个大组件拆成两个小组件）不需要改测试。只要 `data-testid` 还在，测试不关心组件层级怎么变。

### MSW：接口数据的断言

另一类是接口数据的断言。用 MSW 模拟接口返回，构造 mock 数据，验证数据有没有正确渲染到页面上。比如接口返回了 "Skill 1"，页面上就应该能找到这个文本；翻到第二页的时候，请求参数里的 page 应该是 2。

我们写的是组件级的单元测试——每个测试文件对应一个组件，用 MSW 把接口依赖 mock 掉，测试边界就是组件本身。验证的是"给这个组件什么输入，它应该有什么行为"：props 驱动的渲染、用户交互后的状态变化、接口数据到 UI 的映射。

## 实战案例一：Category API SSR 重构

第一个案例是分类接口的 SSR 改造，属于重构场景。Skills Hub 的分类列表原来是前端硬编码的，要改成从后端接口获取，SSR 预取。同时 InstallCard 组件原来每次切 tab 都发一次请求，也要改成 SSR 预取两份数据、本地切换。总共涉及 6 个文件的联动改动。

### Red 阶段

把需求和接口文档给 AI，告诉它 TDD 模式。AI 先只动测试文件，不碰业务代码。新增了 4 个测试用例：

```tsx
// views-index.test.tsx
it('SSR 返回的分类数据正确渲染为 tab 列表', () => {
  render(<ViewsIndex categories={mockCategories} />)
  expect(screen.getByTestId('category-tab-dev')).toBeInTheDocument()
  expect(screen.getByTestId('category-tab-design')).toBeInTheDocument()
})

it('未提供分类数据时不渲染 tab', () => {
  render(<ViewsIndex categories={[]} />)
  expect(screen.queryByTestId('category-tabs')).not.toBeInTheDocument()
})
```

InstallCard 的测试也全部重写——从模拟接口拦截改成直接传 props 验证渲染，因为重构后 InstallCard 不再自己发请求，而是接收父组件通过 SSR 预取的数据：

```tsx
// 重构前的测试：InstallCard 内部用 useSWR 请求数据
server.use(http.get('/api/extensions', () => { ... }))

// 重构后的测试：InstallCard 变成纯展示组件
render(<InstallCard extensions={mockExtensions} />)
expect(screen.getByTestId('install-card-vscode')).toBeInTheDocument()
```

跑一下，大面积红，预期内的。

### Green 阶段

AI 按依赖顺序改 6 个文件：

1. 接口层：新增类型定义和请求函数
2. 常量文件：删掉硬编码的分类列表
3. ViewsIndex：从 props 读取分类数据
4. InstallCard：移除 `useSWR`，改为 props 驱动
5. SSR 页面：`getServerSideProps` 里用 `Promise.all` 并行预取 3 个 API
6. 类型文件：更新 page props 类型

这里有个关键点：InstallCard 从内部发请求变成纯 props 驱动，这种改动如果没有测试保护很容易改漏。但因为测试已经定义了预期行为，AI 只要让测试通过就行，方向不会偏。

### Verify 阶段

`pnpm test:changed`——88 个测试全绿，不到一个小时。

## 实战案例二：Sort Select 新增功能

第二个案例是新增排序 Select，属于功能新增场景。需求是在视图切换按钮旁边加一个排序下拉框，选项有综合排序、收藏数、安装量、名称，选完之后把 sort 参数传给搜索接口。

### Red 阶段

AI 写了 4 个测试：

```tsx
it('排序下拉框存在且默认选中"综合排序"', () => {
  render(<ViewsIndex {...defaultProps} />)
  const select = screen.getByTestId('sort-select')
  expect(select).toHaveValue('default')
})

it('排序下拉框与视图切换共用容器', () => {
  render(<ViewsIndex {...defaultProps} />)
  const toolbar = screen.getByTestId('view-toolbar')
  expect(within(toolbar).getByTestId('sort-select')).toBeInTheDocument()
  expect(within(toolbar).getByTestId('view-toggle')).toBeInTheDocument()
})

it('切换排序发送 sort 参数', async () => {
  render(<ViewsIndex {...defaultProps} />)
  await userEvent.selectOptions(screen.getByTestId('sort-select'), 'installs')
  // 验证 API 被调用时携带了 sort=installs 参数
})

it('切换排序重置页码为 1', async () => {
  render(<ViewsIndex {...defaultProps} />)
  // 先翻到第 2 页，再切排序，验证页码回到 1
})
```

这个案例有意思的地方在于，写测试的过程中就发现了一个问题。我们测试环境里 gate-ui 的 `Select` 组件是 mock 的，用的是原生 select 标签。但原生 select 的 `onChange` 传的是 `ChangeEvent`，而真实的 gate-ui `Select` 的 `onChange` 直接传 value 字符串。如果不在 Red 阶段发现这个差异，等到写实现代码的时候就会踩坑——组件在测试里能跑但真实环境行为不对。所以我们在 Red 阶段就修正了 mock，让它的行为和真实组件一致。

### Green 阶段

改了 2 个业务文件：常量文件新增排序选项，视图组件新增 state、下拉框和接口传参。

### Verify 阶段

`pnpm test:changed`——106 个测试全绿。

## 多 Agent 并行开发

测试覆盖带来的一个额外收益是：可以同时开两三个 AI Agent 改不同模块。

改完代码必须跑过已有测试，写坏了立刻报红，不是靠人肉 review 兜底。测试覆盖健壮之后，每个 Agent 做完改动就跑 `pnpm test:changed`，绿了就说明没有交叉污染。

冲突检测靠两层机制：

- **同文件冲突**：git merge 自动处理
- **跨文件行为冲突**：`test:changed` 基于依赖图分析。如果 Agent A 改了某个组件的 props 结构，而 Agent B 的模块 import 了这个组件，那 Agent B 的测试会被自动纳入 `test:changed` 的执行范围

## 为什么值得做

三个好处。第一，Red 先行，强制想清楚需求。如果直接让 AI 拿需求去写代码，它很可能写偏方向你还不容易发现。先写测试，你必须搞清楚这个功能到底应该做什么，测试就是可执行的需求文档。第二，AI 的改动变得可控。改完代码必须跑过已有测试，写坏了立刻报红。第三，重构有保障。需要抽 hooks、调整组件结构的时候，跑通测试就能合并，不用反复人工验证。

质量方面，这个项目跑下来，提测阶段功能性 bug 非常少，反馈的基本是 UI 细节——间距、颜色这些，不是逻辑错误。另外测试本身就是行为文档，新人接手读测试就知道组件该有什么行为。

## 成本和局限

诚实地说一下成本。测试代码量和业务代码量比大概 1.15:1，有额外投入。但 AI 生成测试很快，这个成本主要是 AI 承担的。

局限方面，这套方案适合需求明确、接口驱动的组件开发和重构。如果需求还在反复变、UI 还没定稿的探索阶段，先写测试反而是负担。纯样式调整也不太适合，CSS 变化很难用断言有效覆盖。

## 总结

用测试来约束 AI，让问题在编码阶段自动被拦住，而不是靠人到后面去发现。写测试看着多了一步，但它省掉的是后面排查 bug、反复返工的时间。
