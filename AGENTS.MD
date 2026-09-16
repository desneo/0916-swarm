## 文档资料地址
docs/requirement.md 原始项目背景信息和输入
docs/SPEC.md   产品规格文档
docs/ARCHITECTURE.md 架构设计文档

## 禁止修改 requirement.md，除非得到我明确指示

## 信息来源优先级
当多个文档存在冲突时，按照以下优先级处理：
用户明确提出的最新要求；
docs/SPEC.md；
docs/ARCHITECTURE.md；
当前 OpenSpec Change 的 design.md；
当前 OpenSpec Change 的 tasks.md；
现有代码实现；
Agent 自己的推测。
发现冲突时：
不得静默选择一个方案；
不得擅自修改上层需求；
必须明确指出冲突；
涉及需求或架构变化时，应暂停并请求确认。

## 生成实施计划前必须：

1. 阅读 SPEC.md。
2. 阅读 ARCHITECTURE.md。
3. 阅读当前 Active Change。
4. 确定本次变更涉及的 SPEC 要求。
5. 确定本次变更涉及的架构模块。
6. 检查实现方案是否违反 ARCHITECTURE.md。
7. 生成 Change Design。
8. Design 确认后才能生成 tasks.md。

## 开始编码前
agent必须：
1.阅读 docs/SPEC.md；
2.阅读 docs/ARCHITECTURE.md；
3.找到当前正在实施的 OpenSpec Change；
4.阅读当前 Change 的 design.md；
5.阅读当前 Change 的 tasks.md；
6.检查与当前任务相关的现有代码；
7.确认实现方案与需求和架构一致。
不得在不了解上述上下文的情况下直接开始大规模修改代码。
未经更新 `ARCHITECTURE.md`，不得引入架构层面的变更。
未经更新 `SPEC.md`，不得修改需求定义。
不得跳过任务，也不得在未明确说明的情况下重新定义任务。
Agent 不能擅自修改SPEC 来让测试通过。

## 编码过程中要遵守的原则
优先实现简单实现便于人理解；
清晰职责；
明确所有权；
明确生命周期；
可测试；
最小必要修改；
复用已有机制。
避免：
复杂语法
过度抽象、过度设计；
猜测性功能；
无关重构；
重复实现；
为未来需求提前设计复杂框架。

## 文档更新
仅在确实发生变化时更新文档，这时候要提醒我确认后再进行文档更新
如果修改了：
产品需求 → 更新 docs/SPEC.md
系统架构 → 更新 docs/ARCHITECTURE.md；
当前 Change 的实现方案 → 更新 design.md；
当前 Change 的执行任务 → 更新 tasks.md。
不得为了形式而创建额外的文档。

## 架构约束
- 生成或修改任何代码前，必须先阅读 `ARCHITECTURE.md.md`，遵循其中的分层、模块边界与接口约定。
- 若实现与架构文档冲突，先停下来指出冲突，不要擅自偏离。
