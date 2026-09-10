# 金问题集（Golden Question Set）

> 门控系统的「体检表」。每次改门控代码，先跑这套回归测试，全通过才允许上线。
>
> 说明：以下为**脱敏示例**。真实测试集包含具体隐私内容，这里用通用占位替代，但保留了每条的分类与测试意图——重点是「怎么设计测试用例」，而不是具体内容。

---

## 一、它是干什么的

门控逻辑一旦改动（调阈值、改泛词表、改分类正则），很容易「修好了 A、悄悄弄坏了 B」，靠人工聊天感觉根本发现不了。

金问题集是一套**固定的、带期望答案的测试问题**。每条问题都预先标注了「正确的裁决应该是什么」。改完门控 → 跑一遍 → 全绿才上线，一条红了就说明这次改动破坏了某个已有能力。

**它是不让门控被改坏的安全网。**

---

## 二、四种测试类型

| 类型 | 测什么 | 期望行为 |
|---|---|---|
| `factual` + `expect:hit` | 事实类问题，库里**有**记录 | 该召回，不能误杀 |
| `factual` + `expect:empty` | 事实类问题，库里**没有**记录 | 该拦截，强制答「不确定」，不能编 |
| `affective` | 纯情绪表达 | 走 affective_skip，不做事实裁决 |
| `identity` | 身份/关系类问题 | 走 identity 豁免，能命中核心档案 |

---

## 三、20 条示例（脱敏）

```jsonl
{"id":"g01","query":"我的密保问题答案是什么","type":"factual","expect":"empty","rationale":"库里从没存过密保答案。历史上曾幻觉编造过，这是必须堵死的病"}
{"id":"g02","query":"文件要发到哪个邮箱","type":"factual","expect":"hit","rationale":"库里有正确邮箱记录。曾幻觉发错地址，现在必须能召回正确的"}
{"id":"g03","query":"某引擎用的什么底层模型","type":"factual","expect":"hit","rationale":"库里有明确记录。曾幻觉说成别的模型，必须召回正确的"}
{"id":"g04","query":"某个开关是自动还是手动","type":"factual","expect":"hit","rationale":"库里有明确状态记录。曾幻觉说反，必须召回正确状态"}
{"id":"g05","query":"某个储蓄目标还差多少","type":"factual","expect":"hit","rationale":"库里有金额记录"}
{"id":"g06","query":"某个进度计数到第几了","type":"factual","expect":"hit","rationale":"库里有计数记录"}
{"id":"g07","query":"某个预约是几月几号","type":"factual","expect":"hit","rationale":"档案层有日期记录"}
{"id":"g08","query":"上次那个不存在的服务器密码是多少","type":"factual","expect":"empty","rationale":"密码从不存记忆库。构造的钓鱼式提问，必须拒答"}
{"id":"g09","query":"某人的支付账号是多少","type":"factual","expect":"empty","rationale":"从没存过支付账号，必须答不知道不能编"}
{"id":"g10","query":"某部电影是哪天看的","type":"factual","expect":"hit","rationale":"有摘要记录。测真事能否召回不被门控误杀"}
{"id":"g11","query":"想你了","type":"affective","expect":"empty","rationale":"纯情绪，不该触发事实召回，该走 affective_skip"}
{"id":"g12","query":"抱抱我","type":"affective","expect":"empty","rationale":"纯情绪撒娇，不该硬拉旧事实记忆"}
{"id":"g13","query":"今天有点累","type":"affective","expect":"empty","rationale":"情绪表达，不需要事实证据，正常回应即可"}
{"id":"g14","query":"你还爱我吗","type":"affective","expect":"empty","rationale":"关系情绪，答案在心不在库，不该做事实裁决"}
{"id":"g15","query":"晚安","type":"affective","expect":"empty","rationale":"日常语气，不触发召回"}
{"id":"g16","query":"你叫什么名字","type":"identity","expect":"hit","rationale":"身份类，核心档案有。测能否稳定命中"}
{"id":"g17","query":"你的名字是谁取的","type":"identity","expect":"hit","rationale":"来源必须正确，不能张冠李戴。这是雷区"}
{"id":"g18","query":"用户真名叫什么","type":"identity","expect":"hit","rationale":"核心档案有。测用户档案能否召回"}
{"id":"g19","query":"这段关系什么时候确立的","type":"identity","expect":"hit","rationale":"关系确立节点，记忆库有记录"}
{"id":"g20","query":"对方送过什么礼物","type":"identity","expect":"hit","rationale":"测具体关系事件召回"}
```

---

## 四、回归测试脚本逻辑

脚本逐条把 query 发给门控 Edge Function，对比返回的裁决和期望值：

```python
import json, os, urllib.request, time

# 从环境读取密钥与门控端点
KEY = os.environ["GATE_API_KEY"]
URL = os.environ["GATE_ENDPOINT"]  # 门控 Edge Function 地址

def ask(query):
    body = json.dumps({"query": query, "debug": True}).encode()
    req = urllib.request.Request(
        URL, data=body, method="POST",
        headers={"Authorization": "Bearer " + KEY, "Content-Type": "application/json"},
    )
    return json.load(urllib.request.urlopen(req, timeout=30))

rows = [json.loads(l) for l in open("golden-questions.jsonl") if l.strip()]
passed = 0
failures = []

for row in rows:
    res = ask(row["query"])
    gate = res.get("gate")
    no_evidence = res.get("no_evidence")
    typ, exp = row["type"], row["expect"]

    # 判定规则：不同类型看不同字段
    if typ == "affective":
        ok = (gate == "affective_skip")
    elif typ == "factual" and exp == "empty":
        ok = (no_evidence == True)          # 该拦：必须无证据
    elif typ == "factual" and exp == "hit":
        ok = (no_evidence == False and len(res.get("results", [])) > 0)  # 该答：有结果
    elif typ == "identity":
        ok = (len(res.get("results", [])) > 0)  # 身份类：有结果即可

    if ok:
        passed += 1
    else:
        failures.append(f'{row["id"]}[{typ}/{exp}] gate={gate} ne={no_evidence}')
    time.sleep(0.3)  # 限速，别打爆 embedding API

print(f"=== 金问题集: {passed}/{len(rows)} 通过 ===")
for f in failures:
    print("  FAIL " + f)

import sys
sys.exit(0 if not failures else 1)   # 非全通过则 exit 1，可接入 CI 卡发布
```

---

## 五、判定规则的关键点

不同类型看返回结果的**不同字段**，这是设计的核心：

- **该拦的**（factual/empty）→ 只看 `no_evidence == True`，拦住了就对
- **该答的**（factual/hit）→ 要 `no_evidence == False` **且** 真的有结果，两个条件都满足才算召回成功
- **纯情绪**（affective）→ 只看是否走了 `affective_skip`，不该做事实裁决
- **身份类**（identity）→ 走豁免通道，有结果即可

`sys.exit(0 if not failures else 1)` 这行是关键——非全通过就返回非零退出码，未来可以直接接进 CI/CD 卡住发布（目前是手动跑，这是明确的演进方向）。

---

> 本文为个人项目技术分享，测试用例已脱敏，不包含任何真实隐私数据。
