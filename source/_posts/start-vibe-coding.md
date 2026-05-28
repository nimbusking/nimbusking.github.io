---
title: 开始vibe coding
tags:
  - vibe coding
  - 大模型
categories: AI
abbrlink: 14f9916b
date: 2026-05-28 11:21:35
updated: 2026-05-28 11:21:35
---

## 前言
罗列，总结一些关于vibe coding过程中的一些经验

<!-- more -->

## 手动openspec

### 确定需求
基本思路，如果不确定需求，尤其是自己完全不了解的领域，
**先与AI进行需求讨论**
- 目标：我想要干嘛，帮我完成需求文档
- 输入：目前有什么，特别是工程目录下面
- 输出：在指定位置输出需求文档prd.md
- 步骤：明确实现别介，注意可以约束创建：*使用提问的方式帮助我确认需求，不要猜测我的意图，不明确的地方必须向我提问*

过程中可以反复打磨修改需求，直至最终确定

### 设计
正常流程肯定是要通过先【概要】后【详设】的过程
目前主流大模型，在第一步沟通过程中，往往就会给你进行了一定程度的概要设计，
如果没有，可以按照下面方式与AI进行提示词沟通，完成概要（preliminary_design）、详细设计（detailed_design）：

概要设计：

```markdown
目标：
根据需求文档生成概要设计文档

输入：
需求文档 doc/prd.md

输出：
概要设计文档 doc/preliminary_design.md

步骤:
根据需求文档的内容，划分出模块，识别模块与模块之间的关系。生成概要设计文档。
不要猜测我的意图。任何不明确的地方都必须都向我提问。
```



详细设计：
```markdown
目标：
根据概要设计文档生成详细设计文档

输入：
需求文档 doc/preliminary_design.md

输出：
详细设计文档 doc/detailed_design.md

步骤:
根据需求文档的内容，划分出模块，识别模块与模块之间的关系。生成详细设计文档。
模块与模块之间尽量保持相互独立，可以独立进行测试。
不要猜测我的意图。任何不明确的地方都必须都向我提问。
```

### 划分任务
这个步骤主要是要对长任务进行划分，因为往往一个需求文档里面如果只用一个任务进行执行的话，往往上下文会变得异常的长。
按模块拆分

参考提示词:
```markdown
目标：
为每个模块划分最小可执行任务

输入:
需求文档 doc/prd.md
详细设计 doc/detailed_design.md

输出:
任务列表
- doc/tasks/<module-name>.md（每个模块对应一个）
- doc/tasks/progress.md（总体进度）

步骤:
根据需求文档和详细设计
为每一个模块生成vibeCoding用的最小任务。
每个模块对应一个 <module-name>.md
用checklist表示子任务是否完成。
progress.md中用checklist表示模块是否已经完成
```


### 实现
同样，如果只用一个agent实现所有字模块的内容，就会变得上下文非常繁琐、非常长。
可以采用多agent配合模式
![多agent协同](14f9916b/MyCapture_2026-05-28_13-10-13.jpg)

这样就形成了，**一个监工agent配合n个编程agent**协同模式进行开发
这个过程中，我们就可以通过下面的提示词来生成具体的监工与编程agent的提示词：

PS：具体细节按照实际实现场景修改即可，如下面的单元测试场景，这个示例是python

```markdown
目标：
生成VibeCoding用的Prompt

输入：
需求文档 doc/prd.md
详细设计 doc/detailed_design.md
任务划分 doc/tasks

输出: doc/implementation_prompt.md

步骤:
阅读输入信息，了解当前要实现的工程
生成doc/implementation_prompt.md作为VibeCoding的起始Prompt

主Agent，用来跟踪整体的进度
主Agent生成子Agent，用来实现每一个模块，并完成测试整个过程不会有人工参与

代码必须有完整的pytest单元测试并通过mypy和ruff检测

生成prompt过程中
如果有任何不明确的地方都必须都向我提问。
```


## 规范驱动-openspec
