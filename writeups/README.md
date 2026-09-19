# writeups

每章一篇文章草稿。按 `templates/chapter.md` 的固定章节写；文风对齐 vLLM 博客：平、准、每个数字带定义，不出现形容词式的性能描述，不解释参数含义（链接文档），不写概念科普。

## 命名

`chNN-<slug>.md`，例如 `ch01-hbm-preemption-cliff.md`。图放在 `chNN-<slug>/` 同名目录下，由 `analysis/plots/` 的脚本生成。

## 完成判定（四条全满足才算一章完成）

1. 有 baseline
2. 瓶颈有证据（profiler 数据，不是推断）
3. 至少一个干预有测量结果——负结果 + 解释同样算完成
4. 有明确的有效边界声明

## 发布

发布渠道与语言待定（见 `plan/08-open-questions.md`）。发布版与仓库版内容一致，仓库版多一节「复现附录」的完整 manifest 引用。
