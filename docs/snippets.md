# 代码摘录（骨架级）

选了两处最能说明「代码当闸门」的函数，**保留了判断结构和部分规则原文，省略了完整规则集**。
它们不是示意代码，是从真实实现里摘的（仅去掉了与本段无关的注释）。

---

## 一、硬闸：`auditHardGates()`

模型评审 Agent可能被"说得通"带偏，所以结构化决策卡在交给评审 Agent之前，先过一遍硬断言。

```js
// 任一不过即强制打回重审，绕过不了
export function auditHardGates(card) {
  const t = String(card || "");
  const fails = [];
  const need = ["【项目结论】", "【核心假设】", "【去水后ROI】", "【隐性成本】",
                "【人工 vs AI 锚点】", "【风险等级】", "【建议动作】"];

  // ① 字段齐全
  for (const k of need) { if (!t.includes(k)) fails.push("缺段:" + k.replace(/【|】/g, "")); }

  // ② 必须算过账
  if (!/\d/.test(t)) fails.push("全文无数字(未算账)");

  // ③ 每个数字得有来源标记
  if (!/(估算假设|来源|R\d|议题原文)/.test(t)) fails.push("关键数字无来源标记");

  // ④ 隐性成本必须做「若…则…」推演
  if (t.includes("【隐性成本】") && !t.includes("若")) fails.push("隐性成本缺'若…则…'推演");

  // ⑤ 禁空话（完整空话词表另有 9 条，此处略）
  if (t.includes("整体良好") || /建议采纳/.test(t)) fails.push("空话(整体良好/建议采纳)");

  // ⑥ 算术闸：所有算式真求值
  fails.push(...checkCardArithmetic(t));
  return { pass: fails.length === 0, fails };
}
```

**为什么这五条是这五条**：每一条都对应一次真实的翻车——

- 缺段：模型会漏写"人工 vs AI 锚点"，导致结论没有对标；
- 无数字：整页全是形容词，读完不知道花多少钱；
- 无来源：出现过一个凭空的"客诉率 1.2%"；
- 缺"若…则…"：隐性成本只写了"可能有风险"，等于没写；
- 空话："整体良好，建议采纳"是模型最爱的收尾，也是评审最没用的句子。

---

## 二、算数闸：`checkCardArithmetic()`

起因是一次真实事故：决策卡里"8 万 ÷ 12 店"被算错，而且**同一个数被重复乘了两次**。模型自己的算术不可靠，评审 Agent只能事后抓错，所以把算式直接求值。

```js
// 抓完整算式 "数 × 数 × … (≈|=) 结果" 与 "数 ÷ 数 (≈|=) 结果"
// 只校验两侧边界完整的算式（避免截取 "5分钟/次×..." 片段误判）
export function checkCardArithmetic(text) {
  const t = String(text || "");
  const fails = [];
  const U = "(?:%|万|千|亿|元|小时|分钟|人|h|天|月|年|店|例|次|颗|台|条|个|单)?";

  // 单位归一：万→1e4 千→1e3 亿→1e8 %→0.01
  const parse = (s) => {
    const m = String(s).match(new RegExp("^(\\d+(?:\\.\\d+)?)\\s*(" + U + ")?$"));
    if (!m) return null;
    let v = parseFloat(m[1]);
    const u = m[2];
    if (u === "%") v = v / 100;
    else if (u === "万") v = v * 10000;
    else if (u === "千") v = v * 1000;
    else if (u === "亿") v = v * 100000000;
    return v;
  };

  // 容差分档：≈ 允许 5%（四舍五入），= 只允许 1%
  // 实测教训：4.925 万写成 4.9 万曾被 0.5% 误杀
  const tolOf = (sym) => (sym === "≈" ? 0.05 : 0.01);

  // ……乘法/除法两条正则，边界规则是不许从片段中间截取（完整正则略）
  // 命中即 push 一条 "算数错: 表达式（实算 X）"
  return fails;
}
```

**这里的设计判断值得说**：

- 为什么容差分两档？因为「≈」是人写的约等于，允许四舍五入；「=」是断言，只允许 1%。一刀切会把正常的约等写成错误。
- 为什么只校验边界完整的算式？因为文本里常有「5 分钟/次 × 3 次」这种片段，从中间截取会判错。
- 为什么不交给 LLM 复算？因为它也会算错，而且错得没有规律。**用确定性代码做确定性的事。**

---

## 三、还有一道：`auditVerdictGate()`

评审 Agent自己的输出也要校验——必须包含「【验收判定】」段，且值只能是"通过 / 不通过"。缺段或写了"基本通过"这类模糊词，本轮验收直接作废重跑。

```js
export function auditVerdictGate(audit) {
  const t = String(audit || "");
  const i = t.indexOf("【验收判定】");
  if (i < 0) return { pass: false, why: "评审 Agent缺【验收判定】段" };
  const seg = t.slice(i, i + 40);
  if (!/通过|不通过/.test(seg)) return { pass: false, why: "验收判定值非法" };
  return { pass: true, why: "" };
}
```

**三层闸门的关系**：模型产卡 → 硬闸（结构与证据）→ 算数闸（数字）→ 评审 Agent（独立视角）→ 评审 Agent格式闸。每一层都能独立否决，且都不需要模型"配合"。
