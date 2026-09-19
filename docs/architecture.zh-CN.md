# HTML2What 规范文档提取引擎设计

> 状态：目标架构与迁移计划，尚未全部实现。
>
> 当前代码基线：Defuddle `0.19.4`，冻结提交
> `d4c4bad0dc96ca31b2383328e38061cc1490db47`。
>
> 本文是分叉后架构、阶段契约和实施顺序的权威说明。README 继续描述当前可用能力；两者不能混为一谈。

## 当前范围

当前只实现 `html2what-engine`：Defuddle 的继承者、网页文档提取引擎及其 NPM 包边界。

浏览器扩展是后续独立项目。本阶段不创建扩展目录、不引入 `chrome.*`、不实现 popup、picker、storage、权限或订阅更新，也不为扩展提前设计实现代码。文中关于扩展的内容只描述未来依赖关系和阶段 F，不代表当前工作范围。

## 1. 目标

把无限、不可控的网页结构转换为有限、可验证的规范文档，再从规范文档导出不同格式。

```text
网页 DOM / 平台 API
  -> 寻找
  -> 预规范化
  -> 过滤
  -> 规范化
  -> 闭集 Document Profile
  -> 导出
  -> CommonMark / GFM / Obsidian / HTML / Plain Text / 未来格式
```

产品层保留两个主要动词：

- `read()`：理解网页，产出规范文档。
- `export()`：表达规范文档，产出目标格式。

Markdown 是一个输出投影，不是系统的中间真相。

## 2. 已确认的架构决策

1. 硬分叉 Defuddle，不从零重复实现已经存在的网页经验。
2. 分叉目标是收编已有行为，不是长期维护一层薄补丁。
3. 先用特征化测试冻结行为，再机械拆分，最后才改变行为和阶段顺序。
4. 核心能力由项目控制，但不追求零依赖；依赖按可靠性、维护成本、包体积和 MV3 兼容性选择。
5. 规则只描述匹配和动作引用，复杂变换由 TypeScript handler 实现，不发明变换 DSL。
6. 全局、生态、平台、站点只是知识组织方式，不进入运行时 schema 成为 `level`。
7. 导出阶段不读取网页、订阅或用户规则；所有映射和降级策略随 exporter 内置并版本化。
8. 平台 API 是可选 source adapter。可用时优先，失败、未授权或数据不完整时回退 DOM 管线。
9. 最终产出两个项目：一个是 Defuddle 的继承者引擎，一个是独立的浏览器扩展。引擎内部是模块化单体，扩展通过 NPM 包使用它。
10. golden 页面、规范输出和失败诊断是核心资产，规则与 handler 可以借助 LLM 生成初稿，但必须通过契约校验和 golden 回归。

## 3. 为什么选择当前执行顺序

最终确认的顺序是：

```text
寻找 -> 预规范化 -> 过滤 -> 规范化 -> 导出
```

Defuddle 的现有代码已经证明，部分语义必须在破坏性过滤前抢救：

- CSS 隐藏的脚注可能被 hidden removal 删除。
- Bootstrap `.alert` 需要先识别为 callout，否则会被杂讯选择器删除。
- KaTeX、MathJax 的可访问语义常存在于隐藏节点中。
- 懒加载图片的真实地址可能只存在于 `noscript` 或 `data-*`。
- 代码语言经常只存在于即将被剥离的 class 中。

但完整规范化不能全部放在过滤前，否则会提前抹掉 `class`、`id` 等过滤指纹，并浪费时间处理最终会被删除的内容。因此把语义抢救和最终闭集化拆成两个阶段。

## 4. 入口准备

入口准备不属于网页语义阶段，只负责建立安全、稳定的工作副本：

1. 读取 metadata、JSON-LD 和当前 URL。
2. 克隆调用方 Document，后续处理不得修改 live DOM。
3. 展开可访问的 shadow DOM。
4. 解析 React streaming SSR 等尚未落位的内容。
5. 记录 source、权限、诊断和性能上下文。

不变量：除显式交互式 picker 外，`read()` 不修改调用方页面。

## 5. 五个阶段的契约

### 5.1 寻找 `find`

职责：确定正文边界、标题边界和需要一并收集的外部内容。

输入：准备后的 Document、URL、metadata、候选 source adapter 结果。

输出：

```ts
interface LocatedContent {
  root: Element
  titleRoot?: Element
  relatedRoots: Element[]
  selector?: SelectorSet
  confidence: number
  diagnostics: Diagnostic[]
}
```

规则：

- 支持自动打分和显式 `contentSelector`。
- 站点 selector 失败必须自动回退打分。
- 用户记忆采用 CSSSelector、TextQuoteSelector、TextPositionSelector 组成的降级链。
- 寻找只确定边界，不删除节点，不把页面结构改写成 Profile。
- 平台 adapter 产出的内容必须进入与 DOM 内容相同的后续契约。

### 5.2 预规范化 `recover`

职责：在删除前恢复脆弱语义，并保护后续不能丢失的结构。

首批 handler：

- KaTeX、MathJax 2/3、原生 MathML、图片公式。
- Prism、highlight.js 和常见带行号代码块。
- 懒加载图片、`picture/srcset`、`noscript` 图片。
- 隐藏脚注、旁注和正文外脚注区。
- GitHub、Bootstrap、Obsidian 等 callout。

不变量：

- 不剥离原始 `class`、`id` 和匹配所需的 `data-*`。
- 不删除非内容节点。
- 可增加内部保护标记，但标记不得泄漏到最终 Profile。
- handler 输出 `exact | reconstructed | failed` 及诊断信息。

### 5.3 过滤 `filter`

职责：从已定位且完成语义抢救的内容中删除非内容。

来源：

- ABP/EasyList cosmetic filter 子集：`host##selector`、`##selector`、`#@#`。
- 通用结构和文本密度启发式。
- Annoyances、Social、Cookie 等生态组件指纹。
- 平台和站点例外。

安全要求：

- 删除必须记录规则、原因、阶段和文本摘要。
- 保护预规范化标出的公式、代码、脚注、表格、媒体等语义节点。
- 大块文本、低链接密度和其他内容证据用于阻止高风险误删，但阈值必须由 corpus 验证，不能宣称绝对安全。
- 过滤只决定“什么不是内容”，不负责发明文档语义。

### 5.4 规范化 `normalize`

职责：把过滤后的内容彻底坍塌到闭集 Document Profile。

动作包括：

- 统一标题、段落、列表、代码、数学、脚注、表格、图片、callout 等结构。
- 将 class 中的稳定语义迁移到 `data-lang`、`data-latex`、`data-callout` 等 Profile 字段。
- 剥离事件属性、无关 class/id/style 和危险 URL。
- 为未知块级和行内结构执行 defined fallback。
- 运行 Profile validator；悬空 handler、非法元素或非法属性不得静默通过。

不变量：规范化不再重新判断正文范围，也不读取站点专属导出策略。

### 5.5 导出 `export`

职责：把已验证 Profile 编译成目标格式。

规则：

- 不访问原网页 DOM。
- 不读取官方网页规则、订阅规则或用户记忆。
- 方言差异是 exporter 内置能力和降级策略，不是外部规则。
- CommonMark、GFM、Obsidian、HTML、Plain Text 分别拥有独立测试矩阵。
- `colspan/rowspan`、callout、数学、脚注等不可无损表达的结构必须有显式降级路径。

## 6. Document Profile

规范文档不只是一段 HTML：

```ts
interface DocumentProfile {
  metadata: DocumentMetadata
  content: ProfileNode
  resources: Resource[]
  diagnostics: Diagnostic[]
  provenance: Provenance[]
}
```

HTML Profile 和 mdast 是 `content` 的两个有限投影，不等同于完整文档。

Profile 规范由一张可审查的行为表定义：

```text
行为 | HTML 投影 | AST 投影 | 允许属性 | 坍塌来源 | 降级路径 | 测试
```

每增加一种行为，必须同时满足：

1. 在 Markdown、编辑器或文档生态中存在对应语义。
2. 定义 HTML 和 AST 投影。
3. 定义来源 handler。
4. 定义各 exporter 的降级路径。
5. 增加 validator 规则和测试。

初步基线验证：当前 209 个 fixture 的 HTML 输出实际只出现 49 种元素和 28 种属性；class 基本限于 callout、脚注回链和代码语言。这说明闭集 Profile 是对现有行为的显式化，不是另造一套与现实脱节的模型。

## 7. 规则与 handler

规则在语义上由两部分组成：

1. 声明式部分：URL/DOM 匹配、selector、参数和 handler 引用。
2. 代码部分：不可避免的规范化处理 handler。

这两部分可以在源码中分目录，但不能当成两个互不相关的产品。一个完整规则必须绑定到具体 handler 和兼容版本，并作为一个联合能力测试和发布。

继承者引擎内部的目标目录：

```text
html2what-engine/
├── src/core/               # 管线、Profile、规则运行时、handler registry
├── src/handlers/           # recover/finalize 的规范化代码
├── src/rules/              # 官方 selector、match、参数和 handler 引用
├── src/exporters/           # commonmark/gfm/obsidian/html/plain
└── fixtures/                # 规则与 handler 的联合 golden 语料
```

规则模型：

```text
match(URL 轴, DOM 轴) + action/handler reference
```

约束：

- 匹配可声明，复杂变换必须是代码。
- handler 引用、handler 版本、选择器语法和指纹冲突在构建期校验。
- handler 注册时声明运行在 `recover` 或 `normalize`；这是调度信息，不是业务层级。
- 执行顺序按阶段、依赖和特异性确定，平局才使用规则包顺序。
- handler 防重入，重复执行不得破坏结果。

规则住所：

| 住所 | 内容 | 更新方式 |
| --- | --- | --- |
| 引擎内置规则 | 官方数据、handler 引用和基线策略 | 与引擎统一版本发布 |
| 外部订阅源 | 仅数据、参数和已知 handler 引用 | 锁定引擎兼容范围并跑 golden |
| 用户记忆 | find selector 集合 | 用户明确保存 |

外部规则只能引用引擎已经提供的 handler。远程内容永远是数据，不能包含可执行代码，以满足 MV3 要求。

## 8. Source adapter

WordPress、Discourse、MediaWiki、Reddit 等平台可以通过 API 获得比渲染 DOM 更完整的内容。

adapter 契约必须声明：

- URL/DOM 指纹和所需 origin。
- 执行上下文及权限要求。
- 登录态和 cookie 策略。
- 数据完整性判定。
- 失败后的 DOM 回退。

adapter 不能直接绕过 Profile。API、站点 extractor 和普通 DOM 最终都必须返回统一的 source document，再进入 `recover -> filter -> normalize`。

## 9. 两个项目的代码组织

```text
html2what-engine/                 # Defuddle 的继承者，模块化单体
├── src/core/                     # read 管线、Profile、validator、规则运行时
├── src/handlers/                 # 规范化处理代码
├── src/rules/                    # 官方声明式规则，与 handlers 配对
├── src/exporters/                # commonmark/gfm/obsidian/html/plain
├── fixtures/                     # 原始页面、Profile 和 exporter golden
└── docs/                         # 架构、Profile 行为表、attribution

html2what-extension/              # 独立 MV3 宿主
├── src/picker/                   # DOM 高亮、点选、扩大/缩小、selector 生成
├── src/popup/                    # 预览和导出交互
└── src/                         # chrome API、storage、订阅更新和权限
```

依赖边界：

- 引擎项目不导入 `chrome.*`，也不包含扩展 UI、storage 或权限。
- 引擎内的 `core`、`handlers`、`rules`、`exporters` 是内部模块，不是必须独立发布的产品。
- 官方 rules 与 handlers 在同一个引擎版本中原子配对。
- 外部规则包只能依赖已发布的 handler ABI，不得注入远程代码。
- exporter 只依赖 Profile，不依赖扩展或宿主 API。
- 扩展项目只负责权限、storage、订阅更新、picker 和 UI 编排，并通过 NPM 引入引擎。
- NPM 发布的第一目标是完整引擎包；是否以后把 handlers 或 exporters 拆成子包，等接口稳定后再决定。

## 10. 分叉与上游纪律

当前仓库保留 Defuddle MIT LICENSE 和原作者版权声明。正式改名或发布前补充 `docs/attribution.md`，记录：

- 来源仓库和 MIT 许可证。
- 冻结 commit。
- 分叉后的主要修改范围。
- 仍参考或移植的上游修复。

Git 操作要求：

1. `origin` 指向本项目 fork。
2. 增加只读语义的 `upstream` remote 指向 Defuddle 原仓库。
3. 机械拆分期间不混入新功能和行为修正。
4. 上游修复按明确 commit 移植，并单独运行 corpus 回归。
5. 引擎项目和扩展项目的发布节奏可以不同，但扩展必须声明兼容的引擎版本范围。

## 11. 当前基线事实

截至冻结 commit：

- 约 2.28 万行 TypeScript。
- 28 个站点或平台 extractor。
- 209 个真实 HTML fixture 和 209 个 Markdown golden。
- 支持 linkedom 与 JSDOM 两套测试后端。
- 构建、类型生成、安全 lint 和 bundle size 门禁存在且可用。
- 当前核心浏览器 bundle 为 89.4 KB gzip，full bundle 为 205.7 KB gzip。

已知结构债务：

1. `parseInternal` 同时承担编排、重试、adapter、过滤、规范化和安全收尾。
2. `removeByContentPattern` 是近千行启发式集合。
3. `createMarkdownContent` 把多个 Markdown 方言策略混在一个函数中。
4. 直接返回 `contentHtml` 的 extractor 绕过大部分完整规范化；返回 `contentSelector` 的 extractor 会重新进入完整管线。
5. `parse()` 在克隆前会修改 live DOM 中的 `srcSet` 和懒加载图片；`removeImages` 也作用于输入文档。
6. fixture 缺失 expected 时测试会自动生成 baseline 并通过，尚未形成严格门禁。
7. 只有少量 fixture 校验中间 HTML，当前 corpus 主要锁定最终 Markdown。
8. 代理测试依赖假域名不可连接，受真实 DNS/网络环境影响。
9. 所有站点 extractor、大量规则和 handler 当前仍静态编入同一个 Defuddle 基线；目标是把它们在继承者项目内部分层，而不是先拆成多个互不兼容的产品。
10. 当前仓库尚未配置 `upstream` remote。

## 12. 实施顺序

### 阶段 A：冻结和加固基线

- 记录 fork 来源、commit 和许可证。
- 配置 upstream remote。
- 修复 fixture 缺失时静默生成 baseline 的行为；更新 baseline 必须使用显式命令。
- 为全部代表性 fixture 增加规范 HTML/Profile golden，而不只锁 Markdown。
- 增加 live DOM 不变测试，覆盖 `srcSet`、`noscript` 和 `removeImages`。
- 把代理测试改成完全本地、可控的 mock server。

退出条件：旧行为可以稳定复现，测试不依赖外网，不会静默接受新输出。

### 阶段 B：机械拆分现有实现

- 提取入口准备、寻找、预规范化、过滤、规范化和导出函数。
- 保持当前调用顺序和输出逐 fixture 不变。
- 将 debug removal、profile timing 和错误恢复变成显式上下文。
- 不在本阶段重排规则、不增加订阅系统、不改 Profile 语义。

退出条件：所有 Markdown 和 Profile golden diff 为零。

### 阶段 C：建立 Profile 契约

- 编写行为表、类型、validator 和 defined fallback。
- 将现有隐式允许元素/属性迁移到显式白名单。
- 将 callout、代码语言、数学和脚注语义从 class/id 迁移到稳定字段。
- 为 exact/reconstructed/failed 和 provenance 建立诊断模型。

退出条件：每次 `read()` 都返回可验证 Profile；非法输出在测试或构建期失败。

### 阶段 D：统一来源和规则边界

- 让所有 extractor/API adapter 进入同一后续管线。
- 在引擎项目内部把站点 selector、过滤数据和 handler 映射移入 `src/rules/`，把规范化代码移入 `src/handlers/`，并在编译产物中绑定二者。
- 实现规则编译、索引、冲突检查和悬空引用检查。
- 优先实现 KaTeX/MathJax、Prism/highlight.js、懒加载图片和脚注家族。

退出条件：引擎的管线不包含站点分支；新增组件家族只需在引擎内部增加 handler 和配套规则，不需要修改扩展。

### 阶段 E：拆分 exporter

- 定义 exporter capability 和降级矩阵。
- 先冻结现有 Markdown 行为为兼容 exporter。
- 再实现 CommonMark、GFM、Obsidian、HTML 和 Plain Text。
- 使用 CommonMark 规范语料验证解析兼容性，使用项目自有 Profile 用例验证映射正确性。

退出条件：exporter 不访问网页规则；每种 Profile 行为都有目标格式结果或显式降级。

### 阶段 F：独立扩展、picker 和订阅

- 建立独立的 `html2what-extension` 项目，通过 NPM 引入完整引擎包。
- 自动模式显示正文和标题定位结果。
- 手动模式支持 hover、点选、扩大、缩小和回退。
- “记住此选择”保存 selector 集合，失效后自动回退寻找阶段。
- 订阅规则锁版本，更新前运行内置 golden 子集。

退出条件：引擎可独立用于 Node、CLI、服务端和其他宿主；扩展只负责宿主能力，规则更新不执行远程代码。

## 13. 质量门禁

每次行为变更至少回答：

1. 改变了哪个阶段的输入或输出？
2. 是通用结构、组件家族、平台还是站点例外？
3. 是否能复用现有 handler？
4. 是否新增或修改 Profile 行为？若是，降级路径是什么？
5. 旧 fixture 是否先失败，再由修复使其通过？
6. 是否同时验证 linkedom、JSDOM 和真实浏览器 DOM？
7. 是否修改 live DOM、引入远程代码或扩大权限？

红线：

- 正文无解释丢失。
- 非显式操作修改当前页面 DOM。
- 运行时静默忽略非法 Profile 或悬空 handler。
- exporter 重新引入站点判断。
- 为单个网页结构发明没有生态对应物的新 Profile 行为。
- LLM 生成的规则或 handler 未经 fixture 和人工 diff 审核直接发布。

## 14. 衡量方式

准确度不能只用“看起来不错”衡量。基准至少记录：

- 内容保留率：正文、代码、公式、表格、图片和脚注是否完整。
- 杂讯率：导航、广告、推荐、弹窗和重复 metadata 的残留比例。
- 结构正确率：标题层级、列表、表格、代码语言和引用关系是否正确。
- 恢复质量：exact、reconstructed、failed 的数量和原因。
- 人工修正成本：是否需要手动选择，修正能否被用户记忆复用。
- 性能：每阶段耗时、总耗时、重试次数和 bundle 体积。

## 15. 尚待验证的设计点

以下内容保留为候选，不写成既成事实：

- Profile 的最终行为数量及 mdast 扩展方式。
- ABP 子集的具体语法边界和是否需要完整广告规则引擎。
- W3C Annotation Selector 的持久化 schema 和迁移策略。
- 规则特异性、显式依赖、排除规则和冲突处理的最终算法。
- 微信 `section` 布局的语义恢复范围和样式白名单。
- 订阅包签名、版本锁定和回滚机制。
- API adapter 在不同浏览器中的 origin 权限、cookie 和 CORS 行为。
- 哪些第三方依赖保留、替换或 vendor。

这些事项应通过最小原型和 corpus 数据裁决，不能仅凭架构偏好决定。
