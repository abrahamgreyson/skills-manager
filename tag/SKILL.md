---
name: tag
description: 给当前仓库（或指定 commit）创建 / 打 git tag 做发版标记。用户说"打个 tag""发版""标版本""version bump""按历史补 tag""标里程碑""semver 怎么 bump""打个版本号"等创建 tag 的请求时用——不要凭印象自己定版本号或猜激进保守。覆盖 SemVer major/minor/patch 判断、annotated tag 格式、HEAD 单 tag vs 按历史补打多 tag。仅限创建 tag：不处理 push tag、删 tag、改 tag message、查某 commit 属于哪个 tag、看两个 tag 之间的 commit；也不含 deploy/publish/merge。
---

# Tag（打 git tag 发版标记）

> 只做一件事：分析变更、算版本、打 **annotated** tag。
> 不含 ship——不 push、不部署、不发包、不 merge。后续会有独立的 ship skill 把 tag 合并进发布流程。

## 何时触发

用户说"打 tag / 发版 / 标版本 / version / 按更新历史打 / 标里程碑 / 给这个版本打个标"等。只要请求落在"git tag"这个动作上就用它，即使没明说 SemVer。

## 版本格式

`vMAJOR.MINOR.PATCH`（带 `v` 前缀），如 `v2.0.0`、`v0.3.1`。

## 版本规则（SemVer + 里程碑拓宽）

判断的不是"改动量"，是**这一版相对上一版，有没有质变**。

### MAJOR（X.0.0）— 质变 / 里程碑

满足任一即 bump major：

- **结构/架构重大变更**：monorepo→单仓、目录大重构、代码组织换范式
- **技术栈变更**：删/换框架（删 Laravel、换前端框架、换状态管理）
- **新增独立服务/应用**：monorepo 加 app、加微服务、加独立可部署模块 ⭐
- **量级新特性**：引入一整个新系统（新特效体系、新页面体系、新核心能力）
- **破坏性变更**：API/行为不兼容旧用法（删接口、改契约）

> **为什么把"新增服务"也算 major**：major 标记的是"用户或外部依赖需要知道这一版和上一版有质的差别"。新增服务是可见的能力扩张，不是内部整理——它值得被钉成一个里程碑，所以纳入。这条是用户明确拓宽的，别漏。

### MINOR（0.X.0）— 向后兼容的新功能

- 新页面、新组件、新配置项、新能力（但不到"新服务/新系统"级别）
- 能力增强、功能扩展

### PATCH（0.0.X）— 不改公共行为

- bug 修复
- 文案、样式、视觉调整
- 清理、重构（不改变对外行为）
- 文档、测试、依赖升级

### 取最高级别

一次发版只 bump 一次，由这批变更里**最高级别的那条**决定。例：既有新功能（minor）又有 bug 修复（patch），发 minor。

## 判断流程

1. **看最新 tag**
   ```bash
   git tag -l --sort=-v:refname | head -1
   ```
2. **通读自上个 tag 以来的所有变更梗概**（无 tag 则看全部）。**必须完整过一遍，不能只看最近几条**——漏掉的 commit 可能正好决定版本级别。
   - commit 梗概：`git log <last-tag>..HEAD --oneline`
   - 文件层面看动了哪里：`git diff <last-tag>..HEAD --stat`（每文件增删行数）或 `git log <last-tag>..HEAD --name-only`
   - **只看梗概 + 文件清单，不读文件全文**——目的是判断"这版变了什么级别的东西"，不是 code review。看 commit message 标题 + 文件路径 + 行数变化通常就够归类。
3. **按规则归类**，取最高级别 → 算出新版本号。
4. 打 annotated tag（见下）。

## tag 节奏

### 默认：HEAD 单 tag

分析当前 HEAD 相对最新 tag 的变更，在 HEAD 打一个最新版本。

### 按历史补多 tag

当用户说"按历史打 / 按更新历史 / 补里程碑 / 给历史版本打标"时，把变更按里程碑分组，对**每个里程碑的收尾 commit**各打一个 tag。

例：HEAD 相对 v1.0 有三组变更——引入新服务（commit A 收尾）、i18n 重构（commit B）、删旧框架（commit C）。分别打 `v1.1.0`(A) / `v1.2.0`(B) / `v2.0.0`(C)。

> 补历史 tag 时版本号必须**按 commit 时间顺序递增**，不能倒挂。先列出"commit → 版本"映射给用户确认，再打。

## 首个版本（仓库无 tag）

看成熟度判断：

- **已上线 / 有真实用户 / 稳定运行** → `v1.0.0`
- **早期 / 还在做 / 未发布** → `v0.1.0`（`0.x` 本身就表示"未正式"）

不确定时倾向 `v0.1.0`——`0.x` 可以随便 bump，正式 `1.0` 之后语义就重了。

## tag 格式

一律 **annotated tag**（`-a` + `-m`），带 message。不要用 lightweight tag（不带 message，丢失发版说明）。

```bash
git tag -a vX.Y.Z -m "vX.Y.Z — <一句话摘要>"
```

message 多行时：

```
vX.Y.Z — <一句话摘要>

- <用户/外部能感知的主要变更 1>
- <主要变更 2>
```

message 写"能感知的变更"，不写内部细节（如"重命名变量""抽函数"）。一条规则：读完 message 能回答"这版到底变了啥"。

## 边界（这个 skill 不做）

- **不** `git push`（含 `git push --tags`）——推送由用户或 ship skill 决定
- **不**部署 / 发布 / 上线
- **不** npm/cargo/pip publish 等任何包发布
- **不** merge PR、改代码、改配置
- **不**用 lightweight tag

打完 tag 就停，告诉用户打了哪些 tag、各自落在哪个 commit，让用户决定后续（push / ship）。

## 示例

**例 1：默认单 tag**
> 用户："打个 tag"
- 最新 `v1.2.0`。`git log v1.2.0..HEAD` 显示：修两个 bug、改文案、加测试。
- 全 patch → **v1.2.1**。
- `git tag -a v1.2.1 -m "v1.2.1 — 文案修订 + 修两个 bug"`

**例 2：按历史补多 tag**
> 用户："按更新历史给我打 tag，可以多个"
- 最新 `v1.0`。HEAD 有三组里程碑：删 Laravel（commit X）、引入 warp 特效（commit Y）、清理过时文档（commit Z）。
- 删 Laravel = 删框架 → major 级；warp = 量级新特性 → major 级；清理 = patch 级。
- 按时间递增分配：`v2.0.0`(X，删框架) / `v2.1.0`(Y，新特性，minor 也行但量级大归 major) / `v2.1.1`(Z，清理)。
  ※ 实际分配时先把"commit → 版本 → 理由"列给用户确认，再打。

**例 3：首版**
> 用户（新仓库）："给这个打个 tag"
- 已上线、有用户 → `v1.0.0`
- 还在做、没发布 → `v0.1.0`
