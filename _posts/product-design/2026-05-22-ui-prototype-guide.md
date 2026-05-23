---
layout: post
title: "什么是 UI 原型图：从需求沟通到产品验证"
date: 2026-05-22 10:00:00 +0800
categories: [Product Design, 产品设计]
tags: [UI, UX, 原型图, 产品设计, 交互设计]
---

在软件产品开发中，很多问题并不是写代码时才出现的，而是在需求还停留在文字阶段时就已经埋下了。比如“用户可以创建订单”“管理员可以审批申请”“系统要支持筛选和导出”，这些描述看起来清楚，但真正落到界面上时，很快就会出现一连串问题：入口在哪里？页面有哪些字段？按钮放在哪里？异常状态怎么处理？点击之后跳到哪里？

UI 原型图就是为了解决这类问题而存在的。

简单说，**UI 原型图是在正式开发前或产品迭代过程中，用来表达产品界面结构、用户操作路径和页面交互逻辑的可视化方案**。它可以是纸上的草图，也可以是 Figma、Axure、墨刀、Pixso 等工具里的可点击页面，甚至可以是一个用 HTML/CSS/JavaScript、React、低代码/无代码平台或 AI 生成工具做出来的可运行 Demo。

它的核心价值不是“画得好看”，而是让团队提前看见产品将如何被使用。

## 一、UI 原型图到底是什么

UI 原型图不是一个固定形态的东西，它更像是一类产物的统称。只要它能回答下面几个问题，就可以被看作 UI 原型图：

- 页面上有什么信息和组件？
- 用户从哪里开始操作？
- 点击、输入、选择之后会发生什么？
- 页面之间如何跳转？
- 正常、异常、空数据、加载中等状态如何呈现？

因此，UI 原型图可以很粗，也可以很精。

早期阶段，它可能只是几张线框图，用方框表示输入框、按钮和列表。这个阶段重点不是颜色和字体，而是确认页面结构、信息层级和主流程。

到了设计评审或研发交付阶段，它可能已经接近最终视觉效果，包含真实文案、组件状态、页面跳转和交互动效。这个阶段重点是确认体验细节和实现边界。

如果需求本身依赖真实数据、复杂交互或响应式布局，原型甚至可以直接用代码实现。此时它不再只是“图”，而是一个可以在浏览器里运行的交互样品。

## 二、UI 原型图用来做什么

### 1. 把抽象需求变成具体界面

文字需求容易产生理解偏差。比如“支持高级筛选”这句话，产品、设计、研发和业务方脑子里可能是四种不同画面。

原型图可以把这种抽象描述变成具体界面：筛选入口在哪里，筛选项有哪些，是否支持多选，筛选后如何展示结果，条件为空时如何提示。只要画出来，讨论就会从“我以为”变成“这里是不是这样”。

### 2. 梳理用户流程

很多产品问题不是单个页面的问题，而是流程的问题。

例如登录、注册、下单、支付、审批、发布内容、配置规则，这些场景都涉及多个页面和状态。原型图可以把页面串起来，明确用户每一步看到什么、能做什么、下一步去哪里。

一个好的原型不只是静态页面集合，而应该能说明用户路径。

### 3. 提前发现交互和逻辑问题

原型阶段越早发现问题，修改成本通常越低。

如果在开发完成后才发现操作路径太长、字段缺失、权限状态没考虑、异常流程断掉，修改成本会明显变高。原型图可以让团队在编码前就发现这些问题，尤其适合暴露以下风险：

- 页面入口不清楚。
- 信息层级混乱。
- 流程分支缺失。
- 表单校验和错误提示不完整。
- 空状态、加载状态、失败状态没有设计。
- 权限、角色、状态变化没有覆盖。

### 4. 帮助团队对齐

UI 原型图是产品、设计、研发、测试、运营和业务方之间的共同语言。

产品经理可以用它说明需求范围；设计师可以基于它完善视觉和交互；研发可以判断实现复杂度；测试可以据此补充测试场景；业务方可以判断流程是否符合实际工作方式。

没有原型时，团队往往围绕文字讨论。文字越长，歧义越多。有了原型，讨论会更聚焦。

### 5. 给研发和测试提供参考

原型图不是替代需求文档，也不是替代设计稿，但它可以让研发和测试更快理解功能。

对研发来说，原型图可以说明页面结构、组件关系、交互状态和跳转逻辑。对测试来说，原型图可以帮助识别主流程、异常流程、边界状态和权限差异。

如果原型图能配合字段说明、状态说明和接口约定，它的交付价值会更高。

## 三、原型图的常见类型

### 1. 低保真原型

低保真原型常以线框图的形式呈现。线框图通常属于低保真原型，但二者不完全等同：低保真强调视觉和交互细节少、修改成本低；线框图强调用线条、方框和占位符表达页面结构。

它适合早期需求讨论，重点是确认：

- 页面有哪些模块。
- 信息优先级是什么。
- 操作路径是否顺畅。
- 主流程是否成立。

常见载体包括纸笔、白板、Balsamiq、Miro、Whimsical、FigJam、墨刀、摹客 RP、Pixso 原型等。

低保真原型的好处是快，缺点是容易让非设计背景的评审者低估最终效果。因此在评审时要说明：这个阶段讨论的是结构和流程，不是最终视觉。

### 2. 高保真原型

高保真原型更接近最终产品界面，通常包含颜色、字体、间距、组件样式、真实文案和交互效果。

它适合设计评审、用户测试和研发交付，重点是确认：

- 视觉效果是否符合产品定位。
- 组件和状态是否完整。
- 页面跳转是否自然。
- 研发是否能按设计还原。

常见工具包括 Figma、Sketch、Penpot、即时设计、MasterGo、Pixso、Mockplus、Adobe XD 等。需要注意的是，Adobe XD 目前处于维护模式，官方说明不再投入新功能、也不会继续发布新功能；除非团队已有存量资产或明确依赖，否则新项目通常不建议把它作为首选工具。

### 3. 可交互原型

可交互原型可以模拟点击、弹窗、切换、页面跳转、动效甚至变量状态。

它适合用于演示和验证体验。例如用户点击“提交”后出现什么反馈，弹窗如何关闭，列表筛选后如何变化，移动端手势如何触发。

常见工具包括 Figma、Axure RP、ProtoPie、UXPin、Justinmind、Origami Studio、Pixso、墨刀等。

如果交互非常复杂，比如存在大量条件分支、变量、真实输入、动态列表或跨设备交互，Axure、ProtoPie、UXPin、Justinmind 这类工具会比普通点击原型更合适。

### 4. 可运行代码原型

当原型需要验证真实体验时，代码原型往往更有效。

比如：

- 响应式布局在不同屏幕下是否合理。
- 表格、筛选、分页、拖拽是否好用。
- 真实数据量下页面是否可读。
- 组件行为是否符合前端实现习惯。
- 某个技术方案是否可行。

这类原型可以用 HTML、CSS、JavaScript、React、Vue、Next.js 等实现，也可以用 Storybook 做组件级或页面级原型。它的优势是真实，缺点是成本比普通原型更高，也更容易让团队误以为已经进入正式开发。

这里说的 HTML 原型，通常不是只写 HTML 标记，而是指 HTML、CSS 和 JavaScript 组合出来的浏览器原型：HTML 负责内容结构，CSS 负责视觉呈现和布局，JavaScript 负责交互行为。

HTML/CSS/JavaScript 原型尤其适合同时追求“精准”和“交互”的场景。它运行在真实浏览器里，可以在目标浏览器和设备上验证字体、字号、间距、响应式布局、滚动行为、表格宽度、固定列、表单控件、hover、focus、disabled 等原生或组件状态，以及 loading 等业务状态。加上 JavaScript 之后，它还可以模拟弹窗、表单校验、Tab 切换、搜索筛选、分页排序、拖拽、路由跳转、接口请求和页面状态变化。

因此，HTML/CSS/JavaScript 原型不只是把界面画出来，而是让界面在浏览器里实际跑起来。它可以同时表达高保真视觉和真实交互，比静态图片或普通点击原型更接近最终产品。不过，这种精准度依赖投入的工程细节：如果只是随手写几个页面，效果未必比设计工具更好；如果使用真实设计系统、组件库、CSS 变量、响应式规则和 Mock 数据，它就可以成为非常接近真实产品的功能性原型。

### 5. 低代码、无代码或 AI 生成原型

在原型语境下，低代码、无代码和 AI 生成工具常介于设计原型和正式开发之间；但它们并不只用于原型，有些工具也可以直接用于发布网站、上线 MVP，甚至构建生产应用。

Framer、Webflow 更适合网站、落地页、营销页面和品牌化 Web 体验；Bubble 更适合带数据库和工作流的业务应用原型或 MVP，也支持响应式 Web 与移动应用；FlutterFlow 是面向 mobile、web、desktop 的可视化开发环境，移动和跨端场景更典型；Figma Make 这类 AI 原型工具适合从文字描述或已有设计快速生成功能性原型、Web App 和交互式 UI。

这类工具的特点是能较快做出可打开体验、可点击、甚至在部分场景可直接发布的 Demo。它们适合验证想法和演示方案，但是否适合长期生产使用，要结合团队技术栈、数据模型、权限、安全、可维护性和迁移成本判断。

## 四、常见工具怎么选

工具没有绝对最好，只有是否适合当前目标。可以按下面的方式选择：

| 场景 | 推荐工具或载体 | 选择理由 |
| --- | --- | --- |
| 需求还不清楚 | 纸笔、白板、FigJam、Miro、Whimsical | 快速发散，方便推翻重来 |
| 画低保真线框图 | Balsamiq、Miro、Whimsical、墨刀、摹客 RP、Pixso 原型 | 关注结构和流程，不被视觉细节干扰 |
| 做高保真 UI 和交付 | Figma、即时设计、MasterGo、Pixso、Sketch、Penpot | 适合设计、协作、标注和研发交付 |
| 后台系统和复杂表单 | Axure RP、UXPin、Justinmind | 支持更复杂的状态、变量和表单交互 |
| 动效和复杂交互 | ProtoPie、Origami Studio、Figma、Axure RP | 适合表达微交互、动效和设备交互 |
| 验证真实技术体验 | HTML/CSS/JS、React、Vue、Storybook | 更接近真实实现，可验证响应式、复杂交互和浏览器实际表现 |
| 快速做网站、功能性 Demo 或 MVP | Framer、Webflow、Bubble、FlutterFlow、Figma Make | 网站发布、业务流程验证、功能性原型、AI 生成原型或 MVP 探索 |

如果团队没有特殊约束，比较稳妥的组合是：

- 早期用白板、FigJam 或 Miro 梳理流程。
- 中期用 Figma、Pixso、即时设计或 MasterGo 做界面和点击原型。
- 复杂后台或复杂表单用 Axure 补充交互逻辑。
- 高风险交互或核心组件用代码原型验证。

## 五、原型图应该画到什么程度

原型图不是越精细越好，也不是越快越好。关键是匹配当前阶段的决策问题。

如果当前问题是“这个需求要不要做”，低保真原型就够了。此时过早追求视觉细节，反而会让讨论偏离业务价值。

如果当前问题是“这个流程是否走得通”，需要把关键页面、跳转、异常路径画清楚。

如果当前问题是“用户是否能理解并完成操作”，需要可点击原型，最好包含真实文案和关键状态。

如果当前问题是“研发能否实现，体验是否真实可用”，则应该考虑代码原型或至少补充技术验证。

一个实用判断是：

> 原型的精细度，应该刚好足以支持当前决策，而不是追求一次性把所有细节画完。

## 六、好的原型图应该包含什么

如果原型要进入评审、研发交付或用户测试，通常应该尽量覆盖这些内容：

- 主要页面结构。
- 核心用户路径。
- 页面跳转关系。
- 关键按钮和操作反馈。
- 表单字段和校验规则。
- 空状态、加载状态、错误状态。
- 权限差异和角色差异。
- 重要文案和提示语。
- 与研发相关的组件、字段、状态说明。

对于复杂系统，还应该补充流程图、状态机图或字段说明。原型图适合表达界面和交互，但不适合承载所有业务规则。把所有规则都塞进原型备注里，后期会很难维护。

## 七、常见误区

### 误区一：把原型图当成最终设计稿

原型图可以接近最终界面，但它不等于最终设计稿。尤其是低保真原型，重点是结构和流程，不是视觉表现。

### 误区二：只画正常流程

很多原型只展示“成功路径”，没有展示失败、取消、返回、超时、空数据、无权限等状态。真实产品中，异常状态往往更影响体验。

### 误区三：页面画得很多，但流程没有串起来

一堆页面截图不等于完整的流程原型。面向评审、研发或测试的原型，至少要说明用户怎么从一个页面走到另一个页面，否则后续角色仍然需要猜。

### 误区四：过早追求高保真

需求还没稳定时就做高保真，很容易造成浪费。高保真会让人关注颜色、图标、间距，却忽略流程本身是否成立。

### 误区五：以为原型可以替代需求说明

原型图能表达界面和交互，但业务规则、数据口径、权限矩阵、接口约定、异常策略仍然需要文档或表格补充。

## 八、总结

UI 原型图的本质，是用较低成本验证产品想法、界面结构和交互路径。

它让抽象需求变得具体，让用户流程变得可见，让团队对齐变得更容易，也让问题尽可能在编码前暴露出来。

选择工具时，不必执着于某个“最强工具”。更重要的是先明确当前要解决的问题：

- 想确认方向，用低保真。
- 想评审体验，用高保真。
- 想验证交互，用可点击原型。
- 想验证真实可用性，用代码、低代码/无代码或 AI 生成原型。

原型图不是为了让产品看起来已经完成，而是为了让团队更早发现哪里还没有想清楚。

## 术语表

- UI：User Interface，用户界面，指用户与软件交互时看到和操作的界面。
- UX：User Experience，用户体验，指用户在完成目标过程中的整体感受和效率。
- 原型图：用于表达产品界面结构、流程和交互逻辑的设计产物。
- 低保真：Low Fidelity，强调结构和流程，不强调最终视觉效果。
- 高保真：High Fidelity，接近最终界面效果，包含较完整的视觉和交互细节。
- 线框图：Wireframe，用简单线条和占位符表达页面结构，常作为低保真原型使用。
- 可交互原型：可以模拟点击、跳转、弹窗、切换、动效等行为的原型。
- HTML 原型：本文中主要指由 HTML、CSS、JavaScript 组合实现、可在浏览器中运行的代码原型。
- MVP：Minimum Viable Product，最小可行产品，用尽量小的投入获得有效反馈和验证性学习。
- Storybook：用于在隔离环境中开发、测试和展示前端 UI 组件与页面的工具。
- 功能性原型：Functional Prototype，指已经具备一定真实交互、逻辑或数据行为的原型，但不一定达到生产系统的工程质量。

## 参考文献

- Figma, "Free Prototyping Tool: Build Interactive Prototype Designs", https://www.figma.com/prototyping/
- Figma, "Free Online Wireframe Tool For Teams", https://www.figma.com/wireframe-tool/
- Figma, "Intro to Figma Make", https://developers.figma.com/docs/code/intro-to-figma-make/
- Adobe, "Adobe XD Learn & Support", https://helpx.adobe.com/support/xd.html
- Digital.gov, "Wireframes", https://digital.gov/guides/research-collaboration/designing/wireframing
- Digital.gov, "Prototypes", https://digital.gov/guides/research-collaboration/designing/prototyping
- Lean Startup Co., "What is an MVP?", https://leanstartup.co/resources/articles/what-is-an-mvp/
- MDN Web Docs, "HTML: HyperText Markup Language", https://developer.mozilla.org/en-US/docs/Web/HTML
- MDN Web Docs, "CSS: Cascading Style Sheets", https://developer.mozilla.org/en-US/docs/Web/CSS
- MDN Web Docs, "JavaScript", https://developer.mozilla.org/en-US/docs/Web/JavaScript
- Axure, "Axure RP - UX Prototypes, Specifications, and Diagrams in One Tool", https://www.axure.com/axure-rp
- Penpot, "The open-source design platform for teams", https://penpot.app/
- Sketch, "Prototyping Tools for Everyone", https://www.sketch.com/prototype/
- Miro, "Free Online Wireframing Tool", https://miro.com/wireframe/
- Whimsical, "Free Low-Fidelity Wireframes", https://whimsical.com/wireframes/
- Balsamiq, "Wireframing with Balsamiq", https://community.balsamiq.com/support/docs/wireframing/
- ProtoPie, "Interactive Prototyping Tool", https://www.protopie.io/
- UXPin, "UX/UI and Prototyping Tool for Designers & Developers", https://www.uxpin.com/
- Justinmind, "Free design and prototyping tool for web & mobile apps", https://www.justinmind.com/
- Origami Studio, "Origami Studio 3", https://origami.design/
- Storybook, "Get started with Storybook", https://storybook.js.org/docs/
- Pixso, "新一代在线原型设计工具 Pixso", https://pixso.cn/prototype-design/
- Mockplus, "Design, Prototype & Collaborate Better and Faster", https://www.mockplus.com/
- MasterGo, "产品原型", https://mastergo.com/product-prototype
- 即时设计, "专业 UI 设计工具，在线协作更高效", https://js.design/
- 墨刀, "墨刀原型", https://modao.cc/feature/prototype/index.html
- Framer, "Build better sites, faster", https://www.framer.com/
- Webflow, "Website Design Tools & Software", https://webflow.com/feature/design
- Bubble, "AI Prototype Generator", https://bubble.io/ai-solutions/prototyping
- FlutterFlow, "Getting Started with FlutterFlow", https://docs.flutterflow.io/
