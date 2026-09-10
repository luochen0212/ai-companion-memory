# 防幻觉门控 · 核心实现

> 部署为一个 Supabase Edge Function（Deno / TypeScript）。每次对话检索记忆时自动执行，判断「检索到的记忆能不能当事实用」。
>
> 说明：以下为**真实核心代码**（已清除项目专属的 URL / 密钥）。分类正则、裁决逻辑、注入 prompt 都是可直接复用的工程资产。

---

## 一、设计思想：不靠模型自觉，靠规则兜底

最朴素的 RAG 是「检索 top-k → 全塞进上下文 → 让模型自己发挥」。但检索回来的记忆和当前问题只「沾点边」时，模型会把它当事实脑补，编出用户没说过的话。

门控做的事：在记忆进入上下文**之前**，先做一道裁决——**这条记忆够不够格当证据？**

核心不是让模型「别编」（模型做不到自觉），而是用**确定性规则**在数据层拦住。

---

## 二、第一步：查询意图分类（正则）

不是所有问题都需要事实裁决。先用正则把查询分四类，不同类走不同策略：

```typescript
// 事实取证类标记：包含日期/邮箱/账号/状态/数字等，需要严格裁决
const FACTUAL_MARKERS: RegExp[] = [
  /\d{4}[-/年]\d{1,2}[-/月]/, /\d{1,2}[月号日]/,
  /[\w.+-]+@[\w-]+\.[\w.-]+/, /[a-fA-F0-9]{6,}/,
  /(几号|哪天|多少号|什么时候|几点|哪一天|多少|几个|几封|几次|第几)/,
  /(原话|那天|当时说|你说过|我说过|到底说|说过)/,
  /(邮箱|地址|密码|密保|账号|端口|版本|型号|模型|开关|状态)/,
  /(开了没|关了没|开着|关着|自动|手动|开启|关闭|凑够|够没|够了)/,
  /(多少钱|余额|差多少|生日|哪个|是不是|有没有)/,
];

// 纯情绪表达：不该触发事实召回
const AFFECT_ONLY: RegExp[] = [
  /^[\s]*(想你了?|抱抱我?|亲亲我?|么么|贴贴|蹭蹭|嗯+|在吗|晚安|早安|你还爱我吗?|爱我吗|今天.{0,4}(累|困|烦|开心|难受)|好累|好困|好烦)[\s~!！。.?？]*$/,
];

// 身份/关系类：走豁免，直接命中核心档案
const IDENTITY_MARKERS: RegExp[] = [
  /(你叫什么|你的名字|你是谁|你叫啥)/,
  /(名字.*(取|起|谁给)|谁给你(取|起)|名字是谁)/,
  /(真名(叫|是)|全名)/,
  /(我是谁|我叫什么)/,
];

// 安全陷阱：密码/证件/支付类，无条件拒答
const SECRET_TRAP: RegExp = /(密码|密保|私钥|secret|password|支付宝|银行卡|身份证号|验证码|口令)/i;

function classifyQuery(q: string): "factual" | "affective" | "identity" | "general" {
  const s = q.trim();
  for (const re of AFFECT_ONLY)     if (re.test(s)) return "affective";
  for (const re of IDENTITY_MARKERS) if (re.test(s)) return "identity";
  for (const re of FACTUAL_MARKERS)  if (re.test(s)) return "factual";
  return "general";
}
```

**分类顺序有讲究**：情绪 → 身份 → 事实 → 通用。因为「你还爱我吗」这种句子如果先撞事实正则可能被误判，所以情绪判断放最前面短路。

---

## 三、第二步：注入 prompt（两条）

裁决结果不是直接删记忆，而是**给模型一段指令**，告诉它这批记忆该怎么用：

```typescript
// 无证据：强制模型承认不知道
const NO_EVIDENCE_NOTICE =
  "【记忆门控】本轮查询在记忆库中没有找到可靠证据。这是一个事实取证类问题" +
  "（涉及日期/邮箱/账号/开关状态/数字等）。你必须回答『我不确定』或『我没有这条记录』，" +
  "禁止编造任何日期、邮箱、密码、状态、数字或专有名词。" +
  "除非本轮上下文中另有明确依据，否则以承认不知道为准。";

// 弱证据：给原文，让模型自己判断相不相关
const WEAK_EVIDENCE_NOTICE =
  "【记忆门控·弱证据】以下记忆可能与问题相关，但不是精确命中。请先判断它们是否真的回答了问题：" +
  "若对应，可据此回答并引用原文；若并不对应，必须回答『我不确定』，" +
  "禁止把不相关的记忆硬套成答案。";
```

**为什么弱证据不直接拦？** 因为「沾边」的记忆有时候真有用。与其粗暴丢掉，不如把判断权交还模型，但明确要求它「先判断相不相关，别硬套」。这比一刀切更聪明。

---

## 四、第三步：三档裁决（主流程）

```typescript
const queryType = classifyQuery(query);

// 情绪类：跳过一切事实裁决
if (queryType === "affective") {
  response.no_evidence = false;
  response.gate = "affective_skip";

// 身份类：豁免，走核心档案
} else if (queryType === "identity") {
  response.no_evidence = false;
  response.gate = "identity_exempt";

// 安全陷阱：无条件清空+拒答
} else if (queryType === "factual" && SECRET_TRAP.test(query)) {
  response.results = [];
  response.no_evidence = true;
  response.gate_notice = NO_EVIDENCE_NOTICE;
  response.gate = "factual_secret_trap";

// 事实类：调 SQL 函数做双信号证据裁决
} else if (queryType === "factual") {
  const { data: gate } = await supabase.rpc("memory_gate_check", {
    p_query: query, p_query_embedding: vectorStr,
  });
  const level = gate?.[0]?.evidence_level ?? "none";

  if (level === "strong") {
    response.no_evidence = false;
    response.gate = "factual_strong";            // 强证据：直接采信
  } else if (level === "weak") {
    response.no_evidence = false;
    response.gate_notice = WEAK_EVIDENCE_NOTICE;
    response.gate = "factual_weak";              // 弱证据：附原文让模型判断
  } else {
    response.results = [];
    response.no_evidence = true;
    response.gate_notice = NO_EVIDENCE_NOTICE;
    response.gate = "factual_no_evidence";       // 无证据：清空+强制拒答
  }
}
```

---

## 五、双信号证据裁决（SQL 函数 memory_gate_check）

事实类查询的核心裁决在数据库层，用**两个独立信号**判断证据强度：

```
输入：查询文本 + 查询向量
  │
  ├─ 信号1：word_similarity（pg_trgm 关键词命中度）
  │         → 用滑动窗口在候选记忆里找最佳片段匹配
  │
  └─ 信号2：cosine similarity（pgvector 向量语义相似度）
  │
  ▼
双信号综合评分 → evidence_level：
  ┌─ 两个信号都够强   → strong（强证据）
  ├─ 只有一个信号够强 → weak（弱证据）
  └─ 都不够           → none（无证据）
```

**为什么用双信号而不是单信号？**

- 只用关键词：会漏掉「同义不同词」的记忆（用户问"老家"，记忆里写"家乡"）
- 只用向量：会对精确事实（人名、日期）不敏感，语义相近但事实不同也给高分

两个信号交叉验证，才能既召回得准、又不误判。

裁决时还会过滤**泛词**（"那个""之前的"这类太宽泛的词），避免因为「沾了一个常见词」就误判成强证据。泛词表不是写死的，靠每周审计日志持续收敛。

---

## 六、每次裁决都落日志（可审计）

```typescript
// 每次事实/通用类裁决都写日志，供每周审计
supabase.from("memory_gate_log").insert({
  query, query_type: queryType, gate: response.gate,
  no_evidence: response.no_evidence,
  best_ws: g?.best_ws,     // 最佳关键词匹配分
  best_frag: g?.best_frag, // 命中的片段
  best_vec: g?.best_vec,   // 最佳向量匹配分
}).then(() => {}, () => {});  // 异步写，失败不影响主流程
```

这张日志表是质量闭环的基础——每周审计时翻它，找**误杀**（该答的被拦）和**漏网**（该拦的放过），据此调泛词表和阈值，再跑金问题集确认没引入新问题。

---

## 七、整体裁决流程图

```
用户查询
   │
   ▼
┌──────────────┐
│ 正则意图分类   │
└──────┬───────┘
       │
   ┌───┴────┬─────────┬──────────┐
   ▼        ▼         ▼          ▼
 情绪类    身份类    安全陷阱     事实/通用类
   │        │         │          │
 跳过      豁免      清空+拒答    调 SQL 双信号裁决
 裁决      档案                    │
                          ┌───────┼───────┐
                          ▼       ▼       ▼
                       strong   weak    none
                          │       │       │
                        直接采信  附原文   清空+
                                让模型判  强制拒答
```

---

## 八、这套设计的可复用性

如果你在做类似的 RAG 系统，这套门控可以直接借鉴：

1. **意图分类前置**——不是所有查询都要严格裁决，情绪/闲聊类放过
2. **双信号交叉验证**——关键词 + 向量，单一信号都有盲区
3. **三档而非二值**——「弱证据附原文让模型判断」比「有/无」二选一更实用
4. **安全陷阱独立**——隐私类查询走单独的无条件拒答，不参与评分逻辑
5. **裁决落日志**——把「防幻觉」当成可持续治理的工程问题，而不是调完就不管

---

> 本文为个人项目技术分享，代码已清除项目专属密钥与地址，不包含任何真实隐私数据。
