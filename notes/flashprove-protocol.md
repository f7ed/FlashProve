# FlashProve 协议走读（面向一般读者）

这份笔记解释 FlashProve 的协议：谁做什么，每一步在证明什么，哪些结论有证明，哪些只有测试或论证。读者不需要 AI 或密码学背景。每个术语第一次出现时都有一句解释。文末有术语表。

**来源和引用。** 全部内容来自这个仓库的文档、源文件头部注释和你自己的运行日志。每条说法后面有一个链接，指向来源文件，冒号后面是行号。例如 [docs/protocol.md:93](../docs/protocol.md) 指该文件第 93 行。三个词在本笔记里有固定意思：

- 「证明了」：在 Lean 里有机器检查过的定理。Lean 是一个 proof assistant（证明助手），它检查用它的语言写的证明。
- 「测试了」：程序跑过，结果符合预期。测试不是证明。
- 「论证了」：只有纸面上的推理，没有机器检查。

文档自己反复强调这个区别。本笔记保留每一处「不是证明」的说明。

**日期。** 本笔记写于 2026-11-23。日志来自 2026-11-22 在 Vast.ai 的一台 H100 PCIe 上做的阶段 3b 运行（[repro/2026-11-21-vast-h100/REPORT.md](../repro/2026-11-21-vast-h100/REPORT.md)）。

---

## 0. 一段话概览

一个 provider（服务方）有一个语言模型。模型的 weights（权重，训练好的参数）要保密。一个 client（客户）发一段 prompt（提示词），收到一段回复。client 想确认：这段回复确实出自一个固定的模型，不是更便宜的或被改过的模型（[docs/protocol.md:16](../docs/protocol.md)）。

FlashProve 的做法分两步。第一步，provider 用 `cp_commit` 对权重做一次 commitment（承诺）。承诺产生一个 32 字节的指纹，叫 commitment ID。第二步，provider 对每个请求用 `cp_prove` 生成一个 proof（证明）。client 用 `cp_verify` 在 CPU 上检查证明。client 不需要权重，也不需要 GPU（[docs/protocol.md:18](../docs/protocol.md)）。

证明说的是：被检查的那些 token（模型处理文字的基本单位，大约一个词或半个词）满足一条 acceptance policy（接受策略）。接受策略有两种：exact match（精确匹配）和 clipped gap（截断差值）。`cp_verify` 接受时最后一行打印 `YES` 或 `WITHIN LIMITS: …`。`WITHIN LIMITS` 说的比 `YES` 少得多（[guide/how-it-works.md:3](../guide/how-it-works.md)）。

协议有两个目标：soundness with binding（可靠性和绑定）和 weight hiding（权重隐藏）。第一个目标是：承诺固定一组权重，关于这组权重的假声明只有很小的概率被接受。第二个目标是：公开文件和证明不泄露权重，除了一份声明过的清单（[docs/protocol.md:20-23](../docs/protocol.md)）。文档说，这两个目标描述的是设计意图，不是已经证明的结论。

---

## 1. 参与方和三个程序

### 1.1 两个参与方

- **provider**：拥有权重，运行模型，生成证明（[guide/terms.md:88](../guide/terms.md)）。
- **client**：发 prompt，收回复，检查证明（[guide/terms.md:19](../guide/terms.md)）。

### 1.2 三个程序

| 程序 | 谁运行 | 在哪运行 | 做什么 |
|---|---|---|---|
| `cp_commit` | provider | 一块 NVIDIA H100 | 对权重做一次承诺 |
| `cp_prove` | provider | 同一块 H100 | 按 batch（批）证明请求 |
| `cp_verify` | client | CPU，不需要权重 | 每次调用检查一个证明 |

来源：[guide/how-it-works.md:9-12](../guide/how-it-works.md)。

`cp_commit` 和 `cp_prove` 从 `prover/` 构建。`cp_verify` 是 C++ 程序，从 `verifier/` 构建（[AGENTS.md:5-8](../AGENTS.md)）。三个程序都建在共享库 `blocks/` 上（[docs/glossary.md:72-75](../docs/glossary.md)）。

### 1.3 每个程序产生什么

**`cp_commit`** 打印 commitment ID，并写两个目录（[guide/how-it-works.md:16-19](../guide/how-it-works.md)）：

- `isk`：secret key directory（秘密密钥目录）。里面有 seed（32 字节的随机种子）、每个权重表的文件和 counter（计数器）。provider 永远不发布、不复制、不恢复它（[docs/glossary.md:588-593](../docs/glossary.md)）。
- `ivk`：public key directory（公开密钥目录）。里面有 key file（`key.bin`）和 leaf file（`leaf/<k>.bin`）。provider 发布它。

provider 还发布 configuration（配置）。配置是三个公开文件（[docs/glossary.md:142-145](../docs/glossary.md)）：

- `policy.txt`：parameter file（参数文件）。它存模型的形状、证明系统的参数和接受策略。名字叫 policy，内容远不止 policy（[docs/glossary.md:445-447](../docs/glossary.md)）。
- `KL.bin`：大部分 quantization scales（量化尺度，见第 2 节）。
- `layout.txt`：证明里每个数组放在哪里。

**`cp_prove`** 对每个请求写一个 proof file（`<name>.proof`）和一个 token file（`<name>.y`）。如果请求失败，它写一个空的 `<name>.failed` 文件（[docs/glossary.md:562-564](../docs/glossary.md)）。

**`cp_verify`** 读配置、`ivk`、request file（`<name>.req`）、token file 和 proof file，加上 client 传入的 commitment ID。它打印一组固定的行，最后一行是判决（[guide/how-it-works.md:25-27](../guide/how-it-works.md)）。调用形式：

```text
bin/cp_verify CONFIG_DIR PUBLIC_KEY_DIR REQUEST_FILE TOKEN_FILE PROOF_FILE -id COMMITMENT_ID
```

一次完整调用跑三个 stage（阶段）。`key` 阶段检查 `key.bin` 里 committer 的证明。`leaf` 阶段把 leaf file 和 `key.bin` 连起来。`request` 阶段检查请求的证明（[verifier/README.md:105-107](../verifier/README.md)）。

### 1.4 request file 由 client 自己构造

request file 里有一个 window（窗口）：$T$ 个 token 的 id，包含 prompt、回复和 filler（填充 token）。文件末尾还有两个数 $r$ 和 $e$，它们说明检查哪些位置。token file 里只有一个数 $y$：窗口之后的下一个 token，或者 0（[verifier/README.md:96-99](../verifier/README.md)）。

client 必须从自己发出和收到的 token 构造 `.req` 和 `.y`。client 不能用证明附带的文件（[guide/how-it-works.md:21-23](../guide/how-it-works.md)）。原因在第 5 节：证明只覆盖请求里命名的范围，范围由谁写就由谁决定。

### 1.5 client 最后信任什么

`cp_verify` 的一个 `YES` 不会自己站住。client 必须信任几样东西，没有任何调用会检查它们（[guide/how-it-works.md:39-41](../guide/how-it-works.md)）：

1. commitment ID。要从一个你信任的第三方拿到，不能从被检查的服务器拿，也不能从证明附带的文件里拿（[guide/known-limits.md:29-35](../guide/known-limits.md)）。
2. 期望的接受策略和它的数字。
3. tokenizer（把文字切成 token 的程序）和 chat template（把对话排成模型输入的模板）。
4. `cp_verify` 自己。文档建议自己构建它（[guide/known-limits.md:32](../guide/known-limits.md)）。

还有三个配置文件。commitment ID 按设计覆盖它们。但 Lean 里关于 `verify` 的可靠性定理是在「给定公开文件」的前提下陈述的。它本身不说「一个 ID 不可能在两组文件下都通过」。所以 client 要像保管 ID 一样保管这三个文件（[guide/how-it-works.md:32-37](../guide/how-it-works.md)）。

一个被接受的证明不说明这些事（[docs/protocol.md:61-68](../docs/protocol.md)）：

- 权重是谁的，模型好不好。provider 可以对任何模型承诺，包括一个很小或退化的模型（[guide/known-limits.md:29-31](../guide/known-limits.md)）。
- 没被检查的 token 怎么样。
- 整数模型和它来源的浮点模型有多接近。
- 回复是否完整。被截断的回复按较短的回复通过验证。client 要自己检查结尾的 end token。
- provider 是否隐藏了权重。`cp_verify` 不检查 mask（掩码，见第 6 节）是否存在。

---

## 2. 整数模型

### 2.1 为什么要有整数模型

证明需要精确的算术。证明里的等式在一个有限域里成立，域里的数是整数。浮点数的舍入没有办法精确地放进这种等式。所以被承诺的模型是一个 integer model（整数模型）：每个运算都在整数上精确定义（[docs/protocol.md:27](../docs/protocol.md)）。

整数模型近似浮点模型，但不等于它（[docs/model.md:8-11](../docs/model.md)）。没有任何证明说两者有多接近（[guide/how-it-works.md:78-80](../guide/how-it-works.md)）。有一次测量（用 Python 实现的整数模型，不是在发布的程序上）：首选 token 在 6.81% 的位置上和浮点模型不同（[guide/known-limits.md:58-60](../guide/known-limits.md)）。

[docs/model.md](../docs/model.md) 是整数模型的参考定义。它定义的函数叫 Model。Model 是 Llama 4 Maverick 的文本 transformer（一种神经网络结构）的整数版本：整数权重，fixed-point（定点）激活值，power-of-two scales（2 的幂次的尺度），非线性函数用查表（[docs/model.md:8-10](../docs/model.md)）。

### 2.2 形状参数

形状参数都在 `policy.txt` 里，都是公开的（[docs/model.md:53-54](../docs/model.md)）。full size（完整规模）是 Llama 4 Maverick 400B 的形状（[docs/model.md:58-68](../docs/model.md)）：

| 符号 | 含义 | 完整规模 |
|---|---|---|
| $T = 2^{\mathrm{lt}}$ | 窗口里的 token 位置数 | 1,024 |
| $L$ | 层数 | 48 |
| $d$ | residual stream（残差流，每层传给下一层的向量）的宽度 | 5,120 |
| $n_h$, $n_g$ | query head 数；key-value head 数 | 40；8 |
| $\mathrm{dh}$ | 每个 head 的维度 | 128 |
| $f$, $\mathrm{fe}$ | 稠密前馈层的隐藏单元数；一个 expert 的 | 16,384；8,192 |
| $\mathrm{NE}$ | routed expert（路由专家）数 | 128 |
| $V$ | 词表大小 | 202,048 |

$\mathrm{NE} > 0$ 时，奇数层是 mixture-of-experts 层（混合专家层，每个 token 只走一个被选中的 expert），偶数层是稠密层（[docs/model.md:70-71](../docs/model.md)）。

### 2.3 权重和尺度（`KL.bin`）

被承诺的权重是一组整数矩阵：每层的 $W_q, W_k, W_v, W_o$（attention），$W_g, W_u, W_d$（前馈），router 矩阵 $W_r$，每个 expert 的三个矩阵，以及 embedding 矩阵 $\mathrm{Emb}$ 和输出矩阵 $W_h$（[docs/model.md:84-91](../docs/model.md)）。没有 bias 向量，也没有 norm 的 gain：gain 已经乘进矩阵里（[docs/model.md:80-82](../docs/model.md)）。

key file 显示：$\mathrm{Emb}$ 以外的每个权重在 $-4{,}096 \ldots 4{,}095$ 内；$\mathrm{Emb}$ 的每个元素是四个有符号字节组成的数。它不显示更窄的范围（[docs/model.md:96-98](../docs/model.md)）。

`KL.bin` 存 32 位无符号整数。每层 13 个（[docs/model.md:103-111](../docs/model.md)）：

- $k_q, k_k, k_v, k_o, k_g, k_u, k_d$：和对应矩阵相乘后右移的位数。右移 $k$ 位就是除以 $2^k$ 再舍入。这就是「2 的幂次的尺度」。
- $M_1, K_1, M_2, K_2$：这一层两个 norm 的指数。
- $\mathrm{sh}$：形成 softmax 查表键的右移位数。
- $\mathrm{rp}$：这一层是否用 rotary 编码（1 或 0）。

之后每个 mixture-of-experts 层再有 3 个：$k_r, k_{gR}, k_{uR}$（[docs/model.md:113-114](../docs/model.md)）。最后一个 norm 的指数 $M_F, K_F$ 和几个 eps 值在 `policy.txt` 里（[docs/model.md:116-118](../docs/model.md)）。

尺度由 provider 用 `scales/` 里的工具从权重算出来。它们公开，而且反映了很多权重矩阵里最大权重的大小。文档声明这一点不隐藏（[docs/glossary.md:240-244](../docs/glossary.md)）。

一个重要的后果：允许范围内的尺度可以把每一层的贡献都舍成 0。缩小 logit 的尺度会让 clipped gap 的限制更容易满足。所以发布或依赖一个 commitment ID 的人，应该读这些值，或者把整数模型跑一遍（[verifier/README.md:265-271](../verifier/README.md)）。文档还说：一个在允许范围内的 scale 文件可以关掉模型的若干块，最多全部关掉（[guide/known-limits.md:31-32](../guide/known-limits.md)）。

### 2.4 四种公开表

Model 读几张整数表。每个表项由一串固定的 IEEE 754 float64 运算算出，每步舍入一次（[docs/model.md:125-128](../docs/model.md)）：

1. **Rotary 表 $C$ 和 $S$**：对位置 $t < T$ 和 $i < \mathrm{dh}/2$，$C(t,i) = \mathrm{rha}(2^{\mathrm{cs}} \cos x)$，$S(t,i) = \mathrm{rha}(2^{\mathrm{cs}} \sin x)$。rotary 编码把位置信息旋转进 query 和 key 向量（[docs/model.md:134-135](../docs/model.md)）。
2. **Softmax 表 $\mathrm{EX}$**：$\mathrm{EX}[u] = \mathrm{rne}(2^{\mathrm{expS}} \exp(-u/2^{\mathrm{rx}}))$。表在第一个 0 之前结束（[docs/model.md:136-137](../docs/model.md)）。
3. **SiLU 表 $T_s$**：键 $c = -32{,}768 \ldots 32{,}767$，值是 $64 \cdot z/(1+e^{-z})$ 的舍入，再 clamp 到 $\mathrm{smax}$。SiLU 是前馈层用的激活函数（[docs/model.md:138-139](../docs/model.md)）。
4. **Sigmoid 表 $\mathrm{Sig}$**：同样的键，$\mathrm{Sig}[c] = 16\,\mathrm{rha}(4096/(1+e^{-c/256}))$。router 用它给被选中 expert 的输入乘一个系数（[docs/model.md:140-141](../docs/model.md)）。

$T_s$ 是 float64 步骤给出的值，不是对真实函数的舍入：在 $\mathrm{eg} = 8$、$\mathrm{smax} = 4{,}095$ 时有 1,744 个表项比真实函数的舍入高 1（[docs/model.md:143-145](../docs/model.md)）。

参数文件存四个 digest（摘要，哈希值）：`tsilu`、`tsig`、`trope`、`texp`。程序的表和这些摘要不符时必须停止（[docs/model.md:154-166](../docs/model.md)）。3b 的 `cp_commit` 日志里有一行确认这件事（它来自 `cp_commit` 内部的一次 planning 调用）：`plan: this program's four public tables match the parameter file's digests`（[cp_commit.log:108](../repro/2026-11-21-vast-h100/logs/stage3b/cp_commit.log)）。如果一个平台的浮点库舍入方式不同，诚实的证明会以 `NO: table-digest-<name>` 结束（[verifier/README.md:307-309](../verifier/README.md)）。

### 2.5 一次前向传播

每个位置 $t$ 的计算（[docs/model.md:184-274](../docs/model.md)）：

1. $x^0_t = \mathrm{Emb}[\mathrm{tok}_t]$。
2. 每层：norm → query、key、value（rotary 作用在 query 和 key 上）→ causal attention（每个位置只看它自己和之前的位置）→ 输出投影加回残差流 → norm → 前馈（稠密层或 mixture-of-experts 层）加回残差流。
3. 最后一个 norm，然后乘 $W_h$，得到 $V$ 个 logit（每个 token 的分数）。
4. $\mathrm{Model}(\mathrm{tok}) = y = \operatorname{argmax}$ of $\lambda_{T-1}$。分数相同时取 id 最小的 token（[docs/model.md:274](../docs/model.md)）。

一个细节对第 5 节重要：**logit gap**。令 $\mathrm{tok}_T = y$，

$$\mathrm{gap}_t = \max_{v < V}\left(\lambda_t[v] - \lambda_t[\mathrm{tok}_{t+1}] + [v < \mathrm{tok}_{t+1}]\right)$$

它等于 0 当且仅当 $\mathrm{tok}_{t+1}$ 是位置 $t$ 的 argmax。它不是 Model 的值，证明的声明用它（[docs/model.md:276-281](../docs/model.md)）。

attention 的中间值 $q, k, v$ 和 norm 的输出都 clamp 到 12 位带符号，即 $\pm 2{,}047$（[docs/model.md:46-47](../docs/model.md)）。

### 2.6 range conditions（范围条件）

Model 是一个 partial function（偏函数）：只在 8 个范围条件都成立的输入上有定义。条件要在所有 $T$ 个位置、所有层、所有 $2L+1$ 个 norm 上成立（[docs/model.md:294-296](../docs/model.md)）。例如：

| 条件 | 值 | 有定义的区间 |
|---|---|---|
| 1 | $V_x + \mathrm{eps}$（norm 的平方和） | $< 2^{78}$ |
| 2 | norm 的商 $w[k]$ | $-32{,}768 \ldots 32{,}767$ |
| 3 | wide query 或 key 项 | $-32{,}768 \ldots 32{,}767$ |
| 6 | gate 键的高位部分 $\mathrm{gh}$ | $-16{,}384 \ldots 16{,}383$ |
| 8 | norm 读的残差流值 | $\lvert x[k] \rvert < 2^{39}$ |

来源：[docs/model.md:298-307](../docs/model.md)。

prover 接受的区间更窄。原因是 prover 把这些值写成 limb（有符号字节）：两个 limb 到 32,639，五个 limb 到 547,599,908,735。在「有定义」和「prover 接受」之间，Model 有定义，但 prover 不做证明，请求得到 `failed`（[docs/model.md:309-312](../docs/model.md)）。

### 2.7 worst-case check（最坏情况检查）

`cp_commit` 在承诺之前，只从权重和尺度出发，给每个 site（范围条件作用的位置）算一个上界。它不抽样任何 prompt：每个上界都应该对所有 prompt 成立。所有上界都在各自的限制内，它才承诺（[scales/worst-case-bounds.md](../scales/worst-case-bounds.md)，「Before `cp_commit` commits」一段）。

这个检查的目的是：确认没有哪个 prompt 能让一个诚实的 prover 因为范围测试而失败。它和权重隐藏有关：一个因为范围而失败的请求，会向发请求的人透露关于权重的信息。可靠性不依赖这个检查：verifier 从来看不到它，每个证明自己重新证明它的值在范围内（同一页，「The check is about whether」一段）。

文档说这些上界为什么成立只是论证，没有被验证的证明（[docs/model.md:314-317](../docs/model.md)；[scales/worst-case-bounds.md](../scales/worst-case-bounds.md) 开头）。矩阵范数的上界用 GPU 上的 64 位浮点库算，这依赖该库按 IEEE binary64 计算经典矩阵乘法，检查只是抽查（同一页第一个要点）。

Lean 里有自己的两个 worst-case check，它们都不是 `cp_commit` 的检查，没有定理把它们和任何程序的检查联系起来（[guide/what-is-proved.md:192-194](../guide/what-is-proved.md)）。

在 3b 日志里，这个检查用了 634.6 秒，按层打印进度，最后两行是「every committed integer lies in its declared range」和「the worst-case check: every bounded site fits」（[cp_commit.log:117-126](../repro/2026-11-21-vast-h100/logs/stage3b/cp_commit.log)）。

### 2.8 从模型文件到整数

[docs/model.md:342-504](../docs/model.md) 第 11 节定义了怎样从模型的 GGUF 文件（一种存放权重的容器格式）得到这些整数矩阵。没有任何证明检查这个转换：证明只关于被承诺的整数（[docs/model.md:344-346](../docs/model.md)）。实际的处理是：解码存储块（Q4_K、Q5_K、Q6_K、Q8_0、F32、F16、BF16），乘上 gain 和常数，取整个张量的最大绝对值定一个指数 $e$，然后 $w = \mathrm{RNE}(2^e \pi)$，保证 $\lvert w \rvert \le 4{,}095$（[docs/model.md:441-449](../docs/model.md)）。

---

## 3. 对权重的承诺

### 3.1 有限域和编码

所有值都是模一个约 64 位素数的数，这个结构叫 field（域）。随机挑战来自建在它上面的一个更大的域（[docs/protocol.md:91](../docs/protocol.md)）。具体地：$p = 2^{64} - 2^{32} + 1$（Goldilocks 域），挑战来自它的三次扩域（[docs/glossary.md:234-235](../docs/glossary.md)）。源码里基域叫 `F`，扩域叫 `E`（[docs/glossary.md:862-863](../docs/glossary.md)）。

**zero-knowledge encoding（零知识编码）。** 权重表的一行从一个向量开始：若干表示权重的数，加上若干随机元素。这个向量定义一个多项式。编码后的行是这个多项式在远多于向量长度的点上的值。这就是 Reed-Solomon code（一种纠错码）。两个不同的向量给出的行在很大比例的位置上不同。但只看一行的任意 $t$ 个值，它们看起来是随机的，不管权重是什么。$t$ 叫 query bound（查询上限），它不超过随机元素的个数，大约等于它（[docs/protocol.md:93](../docs/protocol.md)）。

**column（列）**：所有行在同一个位置上的值（[docs/protocol.md:93](../docs/protocol.md)）。

### 3.2 weight table（权重表）和 Merkle tree

committer 把所有权重写成小的有符号 digit（基 256 的数位），排成一个矩阵 $M$，叫 weight matrix（权重矩阵）。排法是公开的固定规则。列下标 $j$ 分成若干 block（块）。权重表的一行存一个块：所有 $i$ 和块内所有 $j$ 的 $M[i,j]$。所以表的一行不是 $M$ 的一行。committer 给每行加随机元素，编码每行，然后建树（[docs/protocol.md:113](../docs/protocol.md)）。

**Merkle tree（哈希树）**：一棵树，每个 tree leaf（树叶）是一列的哈希，往上每个节点是两个子节点的哈希。prover 只发树根就承诺了所有行。之后它 open（打开）被选中的列：发出列的内容和从叶到根的路径。每片树叶里有一个 salt（随机盐），这样没人能猜一个没打开的列再去验证猜测（[docs/protocol.md:95](../docs/protocol.md)）。

这种「行是码字、列哈希进 Merkle 树、用行的随机线性组合测试」的承诺来自 Ligero 和 Brakedown 两篇论文。在那两篇论文里，行的组合直接发给 verifier；这里它被承诺（见第 4.6 节的 $G_u$），行里带长期存在的随机 tail（尾部），一个 batch 的所有请求在一个共享点上检查（[blocks/iop/foldeq.h:28-36](../blocks/iop/foldeq.h)；[PUBLISHED-WORKS.md:232](../PUBLISHED-WORKS.md)）。

在 3b 日志里，表的形状是：`W has 2^18 columns, 3076096 rows, 386 units`（[cp_commit.log:2](../repro/2026-11-21-vast-h100/logs/stage3b/cp_commit.log)）。建树分 32 份，每两份约 55 秒，共 962.8 秒（[cp_commit.log:128-146](../repro/2026-11-21-vast-h100/logs/stage3b/cp_commit.log)）。

### 3.3 anchor（锚表）和 leaf table（叶表）

一个承诺的第一张权重表叫 anchor。`cp_prove` 从不对 anchor 证明请求（[docs/glossary.md:43-45](../docs/glossary.md)）。

原因是：每个 batch 的证明都要打开表的几列，这消耗表的随机性。所以 anchor 被复制。一个副本叫 leaf table：同样的权重，用新的随机元素重新编码。这个名字和 Merkle 树的叶子无关。每个 leaf table 有一个公开的 leaf file，里面有一个 link proof（链接证明），证明它和 anchor 编码的是同一组权重。batch 只在 leaf table 上运行（[docs/protocol.md:75](../docs/protocol.md)）。

link proof 的做法：committer 用同样的随机系数把 anchor 的行和 leaf table 的行各折叠成一个向量，对两个向量都做承诺，然后打开两张表的随机列。一个 proximity test（邻近性测试，见第 4.6 节）证明两件事：每个折叠向量和它自己那张表打开的列相符；两个折叠向量在权重部分的一个随机点上相等。不同的矩阵对几乎所有系数给出不同的折叠，不同的折叠在几乎所有点上不同（[docs/protocol.md:129](../docs/protocol.md)）。

每个 link 也要打开 anchor 的列，所以 anchor 的 query bound 限制了 leaf table 的个数。leaf table 不能事后添加（[docs/protocol.md:127-131](../docs/protocol.md)）。

在 Lean 和论文的标准术语里，anchor 叫 first table，leaf table 叫 copy，leaf file 叫 entry file（[docs/glossary.md:25-28](../docs/glossary.md)）。`cp_verify` 的 `leaf` 阶段打印的就是 `the entry`。

### 3.4 key file 和 range proof

key file（`ivk/key.bin`）存 anchor 的树根、它的 OOD answers（见 3.5）和 committer 关于 anchor 的一次性证明（[docs/glossary.md:312-314](../docs/glossary.md)）。一次性证明有两部分（[docs/protocol.md:114-115](../docs/protocol.md)）：

1. **range proof（范围证明）**：用 lookup（查表论证，见第 4.3 节）证明每个 digit 在固定的小范围内，由 digit 组成的权重也在范围内。对 embedding 只检查 digit。没有这一步，committer 可以用巨大的域元素，让模型的等式只在模 $p$ 下成立。源码叫这一步 WF（[docs/glossary.md:1023-1024](../docs/glossary.md)）。
2. **one-time test（一次性测试）**：committer 证明表接近一个码字，并且范围证明的声明对编码后的矩阵成立。它用的程序和每个 batch 之后用的「表上的测试」相同，在 anchor 上跑一次。

key file 的形式是一个 proof file：第一条 chain（哈希链，见 4.5）存范围证明和关于表的声明，然后一个 slot、一条共享 chain、一条第二 chain。三条 chain 有自己的 label（标签），都不是 proof file 的标签（[docs/protocol.md:117](../docs/protocol.md)）。

### 3.5 OOD samples 和 OOD answers（record）

proximity test 允许的距离比较大时，一个被承诺的行可能同时接近好几个码字。这叫 list decoding（列表译码）。为了钉住一个消息，verifier 在每个被承诺的消息之后抽一个随机点（坐标在扩域里）。prover 回答一个经过 mask 的元素：消息的元素是一个 multilinear polynomial（多重线性多项式）在 0/1 点上的值，回答就是这个多项式在抽到的点上的值。除了很小的概率，所有符合这个回答的邻近码字编码同一个消息（[docs/protocol.md:101](../docs/protocol.md)）。这叫 out-of-domain（OOD，域外）sample。

权重表也有一个 OOD sample。它的点由表的树根哈希得出。committer 和树根一起发布每行一个经过 mask 的回答，放在 key file 或 leaf file 里。这些回答叫表的 OOD answers（[docs/protocol.md:101](../docs/protocol.md)）。程序和文件格式把它们叫 record：参数文件里的 `record 1`，`result-v1` 行里的 `record=1`（[docs/glossary.md:406-421](../docs/glossary.md)）。

每个 OOD answer 是该行权重的一个带公开系数的和，用一个 pad 掩盖。pad 是程序从秘密 seed 派生的随机值。pad 存在行消息末尾的三个元素里（三次扩域的一个元素占基域三个元素）（[docs/glossary.md:941-944](../docs/glossary.md)）。这些回答永久公开，每个只有一个 pad 保护。谁拿到 seed，谁就能去掉所有回答的 pad，得到权重的精确线性方程（[docs/protocol.md:287](../docs/protocol.md)）。

OOD seeds 不是 provider 的秘密 seed：任何人都能算出 OOD seeds（[docs/protocol.md:81](../docs/protocol.md)）。

OOD answers 让 committer 的 proximity test 能跑到 Johnson radius（Johnson 半径）。Johnson 半径是编码理论里的一个界：在这个距离内，一个词附近的码字个数有限（[docs/glossary.md:299-303](../docs/glossary.md)）。3b 日志里每个 group 行都写 `rule: Johnson radius with slack`（[cp_commit.log:6-51](../repro/2026-11-21-vast-h100/logs/stage3b/cp_commit.log)）。

### 3.6 mask、counter 和「用完」

**mask（掩码）**：一个随机值。程序在发出任何依赖秘密值的东西之前，把它和秘密值合并，或者和秘密值一起编码。在权重表里，mask 就是和每行一起编码的随机元素（[docs/glossary.md:362-365](../docs/glossary.md)）。

一张表的 mask 恰好覆盖一定数量的列，没有余量。超过这个数的列给出权重的精确线性方程。显示这些列的证明仍然通过验证，泄露无法撤销（[docs/glossary.md:335-343](../docs/glossary.md)）。所以程序计数。

**counter（`count.bin`）**：每张权重表一个文件，在 `isk` 里。每当一个证明显示一列，它就增长一个字节，包括开始了但没完成的证明。production build 从不缩短它，操作员也不能（[docs/glossary.md:158-162](../docs/glossary.md)）。

`cp_prove` 在打开任何东西之前，先把这个 batch 将要打开的列数加到磁盘上的 counter。磁盘报告写入完成之后，它才打开（[docs/protocol.md:140](../docs/protocol.md)）。batch 一开始就计数，无论它是否满、是否成功（[docs/protocol.md:75](../docs/protocol.md)）。

3b 日志里 `cp_commit` 打印了这个限制的全部数字（[cp_commit.log:5](../repro/2026-11-21-vast-h100/logs/stage3b/cp_commit.log)）：

```text
cp_commit: the limit: a stored table's random entries bear 44040191 shown columns in all. Of a leaf its own link shows 62 and every batch 64: each leaf serves at most 688127 batches in all. Of the anchor the key's one-time test shows 62 and every leaf's link 62: M = 1 leaves take 124. ...
```

读法：一张表总共可以显示 44,040,191 列。一个 leaf table 的 link 用掉 62 列，每个 batch 用 64 列，所以 $\lfloor (44{,}040{,}191 - 62) / 64 \rfloor = 688{,}127$ 个 batch。anchor 被一次性测试用掉 62 列，每个 leaf 的 link 再用 62 列。

每个 batch 开始前，`cp_prove` 看所有 leaf table 的 counter。只要有一张没有空间再服务一个 batch，它就停。这样，它停不停不依赖于它选了哪张表。承诺不能延长，新承诺有新的 ID，没有证明把两个 ID 连起来（[docs/protocol.md:83](../docs/protocol.md)；[guide/known-limits.md:93-98](../guide/known-limits.md)）。

这个 688,127 是程序自己的计算，不是安全等级。Lean 定理里出现同一个数，但它在那里数的是另一样东西：对单个请求的回复，而不是最多 64 个请求的 batch（[guide/weight-hiding.md:38](../guide/weight-hiding.md)）。「在限制之下就够了」这一点没有对程序证明（row HW-760，[docs/glossary.md:342](../docs/glossary.md)）。

### 3.7 commitment ID 哈希了什么

commitment ID 是 64 个十六进制数字（32 字节）。它是这几样东西的哈希（[docs/glossary.md:134-136](../docs/glossary.md)；[guide/how-it-works.md:32-34](../guide/how-it-works.md)）：

- 三个配置文件的摘要；
- anchor 的名字：树根和形状；
- OOD seeds 和 OOD answers。

按设计，拿着正确 ID 的 client 不可能被塞一个不同的配置（[docs/protocol.md:79](../docs/protocol.md)）。ID 不覆盖 leaf file，也不覆盖 key file 里的证明：`cp_verify` 读到它们时检查它们（[guide/how-it-works.md:36-37](../guide/how-it-works.md)）。

**关于 binding 有什么是证明了的。** 权重的 binding 本身依赖「theorem on codes」（编码理论的一个定理：ePrint 2025/2055 的 Theorem 4.6，Ben-Sasson、Carmon、Haböck、Kopparty、Saraf 的论文的完整版）。程序的 accounting 把它当作给定。Lean 库里证明了一个「据认为可以从该定理推出」的编码陈述（两个词的情形）。该定理自己的形式没有在 Lean 里证明。「库里的陈述可以从论文的陈述推出」以及「程序的参数满足定理条件」都靠阅读，没有证明。这个定理被用到了没有任何 bit 余量的程度（[docs/protocol.md:119](../docs/protocol.md)）。

此外，纸面上关于 binding 的论证覆盖的是一个不含 OOD seeds 的 ID，而程序的 ID 含 seeds。所以 [docs/protocol.md:121](../docs/protocol.md) 明确说：这一页不对程序声称权重的 binding。Lean 证明了的是（[guide/what-is-proved.md:109-125](../guide/what-is-proved.md)）：

- 对 `verify`（Lean 写的 verifier）：任何承诺，不管谁做的，在给定公开文件下，在任何证明之前就确定了一组权重或者没有权重。
- 对一个 Lean 写的 committer（它只模拟 `cp_commit` 的一部分，只在小规模上和 `cp_commit` 比过数值）：确定下来的权重就是放进去的权重，承诺不能打开成别的权重，误差约 $2^{-89}$ 和 $(t + 2^{23})/2^{124}$。
- 没有定理关于 `cp_commit`。

3b 的 commitment ID 是 `2f50a1aa14592d44e0bcf27b1daaa5d5e94c703a95637ff90309bd8cf5f8ae45`（[cp_commit.log:227](../repro/2026-11-21-vast-h100/logs/stage3b/cp_commit.log)）。`cp_verify` 的 `key` 阶段从文件重新算出同一个 ID（[cp_verify.log:3](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)），最后打印 `id: named by the caller and equal`（[cp_verify.log:206](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)）。

---

## 4. 一个请求的证明

### 4.1 请求和它的三个声明

一个 request 要的是证明，不是文字。在通常的用法里，文字已经存在，之后才构造请求。请求是恰好 $T$ 个 token 的序列，加一个位置范围（[docs/protocol.md:29](../docs/protocol.md)）。

位置和 token 都从 0 数。位置 $i$ 检查 token $i+1$。$n$ 个 prompt token 加 $m$ 个生成的 token 时，位置 $n-1$ 到 $n+m-2$ 检查所有生成的 token。如果 client 用生成 token 自己的位置，第一个生成的 token 就没被检查（[docs/protocol.md:31](../docs/protocol.md)）。文字填不满的位置用任意 token 填充。一个证明只匹配一个请求，包括填充，所以两边必须构造同一个请求（[docs/protocol.md:33](../docs/protocol.md)）。

对被承诺的权重，一个证明声明三件事（[docs/protocol.md:35-39](../docs/protocol.md)）：

1. 整数模型在这个请求上用被承诺的权重运行时，它算出的每个值都留在查表的范围内或该步有定义的范围内。
2. 被检查的 token 满足接受策略。
3. 如果最后一个位置 $T-1$ 在范围内，下一个 token 在那里是模型的首选。这一条在任何策略下都成立。

这三件事都只在「每个 segment 的 sub-proof 检查了模型步骤」的程度上成立（[docs/protocol.md:35](../docs/protocol.md)）。

### 4.2 prover 怎样准备一个 batch

`cp_prove` 作为一个服务运行。它对自己做的东西不运行 verifier（[docs/protocol.md:135](../docs/protocol.md)）。它分三步准备一个 batch（[docs/protocol.md:139-141](../docs/protocol.md)）：

1. 收集请求，随机选一张 leaf table。
2. 在打开任何东西之前，把这个 batch 要打开的列数加到该表的 counter。
3. 对每个请求运行整数模型。

每个被服务的请求得到一个 proof file，有五部分（[docs/protocol.md:145-153](../docs/protocol.md)）：

1. header，写明用了哪张 leaf table；
2. first chain（第一条链），这个请求自己的；
3. 这个请求的 slot；
4. shared chain（共享链），batch 里每个 proof file 里逐字节相同；
5. second chain（第二条链），这个请求自己的。

### 4.3 第一条链：segment、trace、sumcheck、lookup、range check

第一条链把请求的声明归约成一个关于权重矩阵 $M$ 的等式。它的输入是 leaf table 的树根和编号、请求的摘要，以及请求本身的字节（[docs/protocol.md:155](../docs/protocol.md)）。

模型被分成 **segment（段）**。每段对应一个 layer group（若干连续的层）、该组的 attention，或者 output head（输出头）（[docs/glossary.md:614-618](../docs/glossary.md)）。对每段，prover 承诺中间值，叫 **trace**。然后 sumcheck 和 lookup 把这段的等式归约成两类声明：关于已承诺数据的 linear claim（线性声明），和关于权重表的声明。请求的 token 作为公开值进入等式。最后一段是输出头，它的等式说被检查的 token 满足策略，并且如果最后一个位置被检查，下一个 token 是首选。下一个 token 明文发送（[docs/protocol.md:157](../docs/protocol.md)）。

两个工具（[docs/protocol.md:97](../docs/protocol.md)）：

- **sumcheck**：一种交互式协议。它把「几百万项的和等于某个值」这个声明，归约成「定义这些项的多项式在一个随机点上的值」这个声明。
- **lookup（查表论证）**：证明一个被承诺向量的每个元素都出现在一张公开的表里。

两者合起来把「这些被承诺的数是模型的一次正确运行」归约成几条 linear claim。每条说：一个被承诺向量的元素的某个已知组合等于某个已知值。沿途发送的值都用随机 mask 掩盖。

**linear claim** 的正式说法：一个给定的线性形式作用在若干 oracle message 编码的消息上，得到一个给定的公开值。源码里叫 `Row`（[docs/glossary.md:905-907](../docs/glossary.md)）。**oracle message**：一组编码后的行，在一棵 Merkle 树下承诺，在一条 prover 消息里发送。verifier 只读它的几列（[docs/glossary.md:938-940](../docs/glossary.md)）。

**lookup 论证和「forest」。** 这里的 lookup 用 product tree（乘积树）。想法是：对一个随机挑战 $\alpha$，

$$\prod_i (\alpha - K[i]) = a \cdot \prod_t (\alpha - \tau(t))^{m_t}$$

$K$ 是 key 向量（要查的值），$\tau(t)$ 是表的行，$m_t$ 是 multiplicity（第 $t$ 行被查到的次数），$a$ 是处理填充的公开因子（[blocks/iop/lkp.h:11-14](../blocks/iop/lkp.h)）。两边都用乘积树算，树不发送；每层树用一个 sumcheck 证明（[blocks/iop/lkp.h:52-60](../blocks/iop/lkp.h)）。multiplicity 以 bit 的形式承诺，每个 bit 一个 plane（平面）；对每个 bit $j$，一棵乘积树乘出所有该 bit 为 1 的行，只对树根做平方（[PUBLISHED-WORKS.md:209](../PUBLISHED-WORKS.md)）。key 向量本身不发送：调用方知道怎样从已发送的向量读出它（[blocks/iop/lkp.h:16-18](../blocks/iop/lkp.h)）。

`cp_verify` 的源码把 key 一侧的一组乘积树叫 **forest**（森林）（[verifier/src/vf_lookup_argument.h:9](../verifier/src/vf_lookup_argument.h)）。这就是 3b 日志里每个 sub-proof 一行 `8 lookup calls in one forest, 2 more in the smallness test's` 的意思（[cp_verify.log:71](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)）。`cp_verify` 对请求的 sub-proof 里的 lookup 有自己的第二套实现；`blocks/iop/lkp.h` 的实现它只用在 key file 的范围证明 WF 上（[blocks/iop/lkp.h:6-9](../blocks/iop/lkp.h)）。

这个 lookup 的来源有多篇论文：等式本身来自 Papini 和 Haböck（ePrint 2023/1284）以及 Arya（ePrint 2018/380 附录 B.5）；乘积树用 sumcheck 证明来自 GKR 协议的 Thaler 形式；multiplicity 的 bit 是 bit 这件事用 HyperPlonk 的 zero-check（[PUBLISHED-WORKS.md:207-213](../PUBLISHED-WORKS.md)）。

**range check（范围检查，程序叫 `smallness`）。** 等式在模 $p$ 下证明，但声明关于整数。所以每段包含一个范围检查。trace 承诺之后，verifier 抽一组因子，每个是 $-1$、$0$ 或 $1$。一个关于 digit 的 lookup 证明 trace 的值用这些因子组合后是小的。除了很小的概率，值本身也就是小的，在一个更宽的界内。有了这一点，再加上模型数字大小的条件，模 $p$ 下成立的等式在整数上也成立。`cp_verify` 测试公开参数是否满足这类条件，叫 no-overflow conditions（不溢出条件）。「程序的条件足以保证程序的等式」这一点没有证明（[docs/protocol.md:159](../docs/protocol.md)）。它不是 3.4 节的 range proof。源码叫它 RNG 或 `tail`（[docs/glossary.md:966-969](../docs/glossary.md)）。

prover 把每段的承诺折叠成一个短承诺，带到下一段，这样它不用保存整个 trace。最后一个短承诺上跑一次 proximity test（[docs/protocol.md:161](../docs/protocol.md)）。

第一条链结束时，所有关于权重表的声明合并成一条（[docs/protocol.md:163-167](../docs/protocol.md)）：

$$g \cdot M \cdot e + \mu = v$$

$g \cdot M \cdot e$ 是对所有 $i, j$ 求和 $g[i]\,M[i,j]\,e[j]$。verifier 知道因子 $g$、$e$ 和值 $v$。$\mu$ 是 prover 在链开始时承诺的 mask 的一个已知组合。prover 现在承诺 **request vector（请求向量）** $w = g \cdot M$，每个 $j$ 一个元素。链以一个 seal（封印，链末尾的摘要）结束。

### 4.4 slot 和 batch tree

共享的挑战不能在每个请求都固定了自己的声明之前抽取。所以 batch 里所有请求的 seal 成为一棵小的带盐 Merkle 树的叶子，叫 **batch tree**（源码叫 binding tree）。树根进入共享链的输入。一个请求的 **slot** 是它在树里的位置。proof file 里存它的 slot、它那片叶子的盐和路径。请求被放在随机的 slot 里，没用的 slot 放随机摘要。这是为了让 batch 的大小和其他请求不出现在 proof file 的字节里。但一个 batch 用的时间会暴露它的大小（[docs/protocol.md:169](../docs/protocol.md)）。

这个做法的来源：几个 verifier 共享从他们的 transcript 的 Merkle 树根抽出的挑战（Zhang 等，USENIX Security 2022）；带盐的树的形式来自 Batch Signatures（ePrint 2023/492）（[PUBLISHED-WORKS.md:265-266](../PUBLISHED-WORKS.md)）。

3b 日志里 `result:` 行有 `leaves=64 place=58`（[cp_verify.log:164](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)）：batch tree 有 64 片叶子，这个请求在第 58 个位置。源码和打印行把 slot 叫 `place`（[docs/glossary.md:988-991](../docs/glossary.md)）。

### 4.5 哈希链、Fiat-Shamir 和 proof of work

证明是一个文件，没有人实时发挑战。prover 维护一个运行中的哈希，叫 **chain（链）**。这就是 Fiat-Shamir 变换。每个挑战是链的上一个值和 prover 最新消息的哈希。所以 prover 在固定要被测试的内容之前，不可能知道挑战。一个 **round（轮）**是 prover 的一条消息加上随后的挑战（[docs/protocol.md:103](../docs/protocol.md)）。

一条链从它的输入的哈希开始。输入有三部分：参数（含配置摘要）、32 个随机字节、说明在证明什么的项。每条链还有一个整数 **label**，说明这条链跑哪个协议。label 进入第一个哈希和每一轮的哈希（[docs/protocol.md:105](../docs/protocol.md)）。

**proof of work（工作量证明）**：在选定的轮之前，prover 必须做一次哈希搜索，找一个数让哈希的若干 bit 为 0。这让恶意 prover 每次重试一轮都更贵（[docs/glossary.md:492-496](../docs/glossary.md)）。`prover/examples/` 下所有参数文件都用同一个 default setting（默认设置）。它要求在每一轮抽样位置之前做 proof of work（[docs/protocol.md:107](../docs/protocol.md)）。

要注意一点：proof of work 计入 target，但它**不降低**一个假声明躲过一次挑战的概率。它只让每次重试更贵（[guide/terms.md:84-87](../guide/terms.md)）。

3b 日志里 `hf: work bits: pow=20 pows=28 powc=32`（[cp_commit.log:4](../repro/2026-11-21-vast-h100/logs/stage3b/cp_commit.log)）：请求自己的轮 20 bit，batch 共享的轮 28 bit，committer 的轮 32 bit（对应 [cp_verify.log:169](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log) 的 `positions 20 / 28 / 32`）。

哈希用标准 BLAKE2b。另有一个建在 BLAKE2b 压缩函数上的第二个固定哈希函数，它把九个摘要送进证明的 statement。它在 BLAKE2b 已发表的分析之外（[guide/how-it-works.md:117-119](../guide/how-it-works.md)）。

### 4.6 共享链：一个 batch 的请求共享一次打开

每个请求都通过自己的向量 $w$ 对同一个巨大矩阵 $M$ 做声明。$w = g \cdot M$ 必须对照被承诺的表检查。如果每个请求都打开一次表，表的随机性会快得多地用完。所以一个 batch 只打开一次表，所有请求共用（[docs/protocol.md:171](../docs/protocol.md)）。

verifier 抽一个新的随机点，对所有请求相同。一个固定公式把这个点变成因子 $e^*$，每个 $j$ 一个。如果某个 $w$ 不是 $g \cdot M$，那么对几乎所有点，$w \cdot e^*$ 和 $g \cdot (M \cdot e^*)$ 不同。所以一个向量 $M \cdot e^*$，叫 **shared vector（共享向量）**，服务所有请求。共享链证明共享向量是正确的（[docs/protocol.md:173](../docs/protocol.md)）。源码里这个共享向量叫 $\Omega$（[blocks/iop/lin.h:15-17](../blocks/iop/lin.h)）。

共享链的输入是 leaf table 的树根和 batch tree 的树根（[docs/protocol.md:175](../docs/protocol.md)）。它有五步（[docs/protocol.md:177-183](../docs/protocol.md)）：

1. prover 用块因子折叠表的行，承诺结果，叫 **folded vector（折叠向量）**。源码叫 $G_u$。
2. verifier 抽样位置，prover 打开表在那些位置的列。
3. 任何人都能用块因子折叠一个打开的列。所以每个打开的列给出关于折叠向量的一条 linear claim。这些声明把折叠向量和表绑在一起。OOD answers 用同样方式折叠，再给一条。
4. prover 从折叠向量导出共享向量，用块内偏移的因子，并承诺它。这是两个向量之间的线性关系。
5. proximity test 证明关于这两个向量的所有 linear claim。

这是一个 batch 里唯一打开 leaf table 的步骤。共享链加一个请求的第二条链合称 **the test on the table（表上的测试）**。3.4 节的一次性测试是同一对链，在 anchor 上跑一次。两者不能互换：一个 batch 的测试只对已经通过过一次测试的表可靠（[docs/protocol.md:185](../docs/protocol.md)）。

[blocks/iop/lin.h](../blocks/iop/lin.h) 头部给出了同一件事的代数形式。请求 $c$ 声明 $\langle g_c \otimes \mathrm{eq}(\psi_c, \cdot),\, M\rangle + (\text{mask}) = \tau_c$。三部分：第一部分每个请求发 $W_c = g_c^{\mathsf T} M$；共享部分 verifier 抽共享点 $\psi^*$，prover 发 $G_u$ 和 $\Omega = M\,\mathrm{eq}(\psi^*, \cdot)$；第二部分把 $\langle g_c, \Omega\rangle$ 和 $W_c$ 在 $\psi^*$ 处绑定（[blocks/iop/lin.h:6-21](../blocks/iop/lin.h)）。文件头部说，「请求向量和共享表之间的绑定」这两条检查的形式来自 Greyhound（ePrint 2024/1293），只是形式：Greyhound 用格承诺，这里用码字上的哈希树（[blocks/iop/lin.h:23-26](../blocks/iop/lin.h)；[PUBLISHED-WORKS.md:242](../PUBLISHED-WORKS.md)）。

共享向量不发送而是承诺，这一点和 Greyhound 的「不发送 prover 的回应」以及 Hachi（ePrint 2026/156）的做法相关（[PUBLISHED-WORKS.md:243](../PUBLISHED-WORKS.md)）。

### 4.7 第二条链

第二条链把请求的声明和 leaf table 绑在一起。它的输入是两个 seal。一个 proximity test 证明两个等式成立：第一条链声明的 $w \cdot e + \mu = v$，以及 $w \cdot e^* = g \cdot (M \cdot e^*)$（[docs/protocol.md:187](../docs/protocol.md)）。

### 4.8 proximity test：fold → code switch → 递归

verifier 只看几列。所以它需要知道，被承诺的东西 **接近** 一个 codeword（码字）：它和某个向量的编码只在少于固定比例的位置上不同。**proximity test** 证明这一点。prover 用随机因子把行合并成一行，叫 **fold（折叠）**。它把这行切成更短的行，在更短的码下承诺它们，叫 **code switching（换码）**。verifier 在随机位置上对照旧承诺检查新承诺。这个过程重复，直到剩下的东西短到可以带 mask 发送。这个测试同时证明关于向量的 linear claim：它们被合并成一条，fold 把它带着走，verifier 最后对照发送的内容检查（[docs/protocol.md:99](../docs/protocol.md)）。

[blocks/iop/iopp.h](../blocks/iop/iopp.h) 头部给出 block（模块）的顺序（[blocks/iop/iopp.h:6-18](../blocks/iop/iopp.h)）：

| block | 做什么 |
|---|---|
| BAT | 把 group 上所有未决的 linear claim，加上每个 oracle message 的 OOD sample 的声明，合并成一条。方法：第 $i$ 条声明乘 $x^i$ 再求和，$x$ 是一个挑战。prover 不发任何东西（[blocks/iop/bat.h:6-10](../blocks/iop/bat.h)） |
| FOLD（每个 stage 一次，最后一个除外） | 用带 hiding polynomial（隐藏多项式）的 sumcheck，把 $2^k$ 个主行和 $2^h$ 个 mask 行折叠成三个单行：主行、旧 mask 行、一个新 mask。verifier 在这里不测试任何等式（[blocks/iop/fold.h:6-23](../blocks/iop/fold.h)） |
| CS（code switch） | 把这三个单行在新的码下重新编码成一个新的 oracle message $g$。新消息里存旧行的消息和它们的编码随机数。旧行在位置 $x$ 的符号是新消息的一个公开线性形式。verifier 在 $t$ 个随机位置读旧行，每个读到的符号变成一条声明。如果旧行和新消息的编码相差超过旧 group 的距离，某条声明为假的概率至少 $1 - (1-\mathrm{distance})^t$（[blocks/iop/cs.h:6-24](../blocks/iop/cs.h)） |
| BASE（最后一个 stage） | 一个主行和一个 mask 行带 mask 发送。verifier 检查声明，并在一组位置上检查编码 |

一个 **stage** 是一组行和它们的码。有 $n$ 个 stage 的测试跑 $n-1$ 次 FOLD，每次带一个 code switch，最后一个 stage 进 BASE（[docs/glossary.md:999-1001](../docs/glossary.md)）。**group**：一组完整的 oracle message，在同一个码下，被一个 proximity test 在同一组位置上读（[docs/glossary.md:876-877](../docs/glossary.md)）。

这个递归的来源是 Ligerito（ePrint 2025/1187）第 6.1 节：prover 把 verifier 本来会收到的向量作为一个新矩阵承诺，用同一个协议证明 verifier 对它的内积检查，然后重复。随机系数合并 linear claim 并用一次 sumcheck 证明来自它的第 4.3 节。区别：那里系数独立抽取，这里是一个挑战的幂；这里每个 stage 有 mask 行，新消息还带旧行的编码随机数，每个新消息有自己的 OOD sample（[PUBLISHED-WORKS.md:237](../PUBLISHED-WORKS.md)）。换码本身沿用 Orion（ePrint 2022/1010）第 4.2 节和 Ron-Zewi–Rothblum 的「code switching」（[PUBLISHED-WORKS.md:239](../PUBLISHED-WORKS.md)）。Ligerito 自己说它的协议结构上和 WHIR 相似（[PUBLISHED-WORKS.md:238](../PUBLISHED-WORKS.md)）。

整个 proximity test 是 Chiesa、Fenzi、Weissenberg 的论文「Zero-Knowledge IOPPs for Constrained Interleaved Codes」（ePrint 2026/391）的测试，带一些差异。`blocks/` 实现了它的零知识 sumcheck、code switch 和最后一步（[PUBLISHED-WORKS.md:9](../PUBLISHED-WORKS.md)；[guide/how-it-works.md:111-113](../guide/how-it-works.md)）。IOPP 是 interactive oracle proof of proximity（邻近性的交互式预言机证明）的缩写（[blocks/iop/iopp.h:9](../blocks/iop/iopp.h)）。

3b 日志里 `cp_commit` 打印了每个测试的 stage 结构。例如 batch 里对折叠后的表（[cp_commit.log:14-18](../repro/2026-11-21-vast-h100/logs/stage3b/cp_commit.log)）：

```text
group: folded table: code 2^24, message 4194304 + 101 random, 2^7 rows; mask code 2^13, 64 + 152; distance 0.497247, m = 182, list 91, 101 positions; rule: Johnson radius with slack; ...
group:   stage 2: 2^5 rows of 131072 + 54 in 2^21; 2^2 mask rows of 256 + 54 in 2^12; list 222, 54 positions
group:   stage 3: 2^4 rows of 8192 + 54 in 2^17; ...
group:   stage 4: 2^4 rows of 512 + 54 in 2^13; ...
group:   stage 5: 2^0 rows of 512 + 51 in 2^14; ...
```

读法：第一个 stage 的码长 $2^{24}$，消息长 4,194,304 加 101 个随机元素，128 行；每个 stage 的行变短、变少；每个 stage 抽 54 或 51 个位置。「+ 101 random」就是 3.1 节的随机元素，它们决定 query bound。

### 4.9 OOD samples 在证明里

每个被承诺的消息之后（除了某一步的 mask），verifier 都抽一个 OOD sample，prover 回答一个带 mask 的元素（[docs/protocol.md:101](../docs/protocol.md)）。源码里做这件事的 block 叫 SEND（[docs/glossary.md:985-987](../docs/glossary.md)）。回答的 mask 叫 pad（[docs/glossary.md:941-942](../docs/glossary.md)）。SEND 和 BAT 是 code switch 可靠性证明的第一步和最后一步（[PUBLISHED-WORKS.md:20](../PUBLISHED-WORKS.md)）。

### 4.10 为什么假声明应该被拒绝（设计论证）

[docs/protocol.md:262-269](../docs/protocol.md) 给出直觉，文档明确说这是草图，不是证明：

- 挑战测试的一切都在挑战存在之前固定（Fiat-Shamir）。所以有假声明的 prover 需要运气。每一轮只给它很小的机会，一次尝试的总机会最多是各轮机会之和。
- 哈希树让 prover 不能在承诺之后改答案。
- proximity test 显示承诺接近一个码字；在允许的距离上可能有几个码字接近，OOD sample 只留下一个消息。这一点依赖 theorem on codes。
- range proof、range check 和模型数字大小的条件，防止等式只在模 $p$ 下成立。
- 表上的测试把请求的声明绑到 leaf table，link proof 把 leaf table 绑到 anchor，commitment ID 固定 anchor 的树根和配置。

作弊者可以换消息重试。每次尝试要一次哈希调用，所以界随哈希调用数增长。

这些论证的完整形式在项目的一份 technical report（技术报告）里。它不公开、未经同行评审、不在仓库里。读者无法核对文档对它的转述。文档说两个论证都没有被检查到可以依赖的程度，而且没有任何规模的一套完整参数被确认满足它的全部条件（[docs/protocol.md:260](../docs/protocol.md)）。

---

## 5. 接受策略

### 5.1 范围：`r` 和 `e`

一个请求是 $T$ 个 token 的窗口加两个数 $r$ 和 $e$。位置 $t$ 预测 token $t+1$，所以范围 $r$ 到 $e-1$ 检查 token $r+1$ 到 $e$。对 $P$ 个 prompt token 和 $R$ 个回复 token，$r = P-1$，$e = P+R-1$。范围之外什么都不覆盖（[guide/how-it-works.md:45-50](../guide/how-it-works.md)）。

如果 $e = T$，调用还检查 $y$，窗口之后的那个 token。token file 存 $y$，否则存 0（[verifier/README.md:96-99](../verifier/README.md)）。

3b 的请求是 24 个 token 的编造 prompt 加 1,000 个 token（[REPORT.md:138](../repro/2026-11-21-vast-h100/REPORT.md)），所以 $r = 23$，$e = 1024 = T$。`covered:` 行写 `positions=23..1023 tokens=24..1023 y=yes`（[cp_verify.log:209](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)）。`y=yes` 表示 $y$ 也被检查了。`next-token=no` 表示不是只检查 $y$（[verifier/README.md:122-125](../verifier/README.md)）。证明的答案是 token 52599（[full-f2.out:14](../repro/2026-11-21-vast-h100/logs/stage3b/full-f2.out)）。

### 5.2 exact match

在 exact match 下，`YES` 的意图是：每个被检查的 token 是被承诺整数模型在它位置上的 top choice（首选，logit 最大的 token，相同时取 id 最小的），在误差界之内（[guide/how-it-works.md:57-59](../guide/how-it-works.md)；[docs/glossary.md:735-736](../docs/glossary.md)）。这相当于 greedy decoding（贪心解码：每一步都选最高分）（[docs/protocol.md:43](../docs/protocol.md)）。采样不在任何策略下被检查，因为请求不带随机种子。

参数文件的三行决定策略的种类：`C`、`theta`、`cap` 读作 256、256、0 就是 exact match（[guide/what-is-proved.md:284-286](../guide/what-is-proved.md)）。3b 的 `covered:` 行里正是 `C=256 theta=256 cap=0 … policy=exact-match`。

### 5.3 clipped gap

一个 token 的 **score gap**：它的 logit 比该位置最大 logit 低多少，如果首选的 token id 比它小再加 1（[docs/glossary.md:583-585](../docs/glossary.md)）。它的 **cost**：score gap 加常数 $2^{\mathrm{sh}}$，所以首选 token 的 cost 是 $2^{\mathrm{sh}}$，不是 0（[docs/glossary.md:151-153](../docs/glossary.md)）。

clipped gap 对所有被检查 token 一起设两个上界（[guide/how-it-works.md:61-64](../guide/how-it-works.md)）：

- 最多 `cap` 个 cost 可以超过 `C`；
- 每个 cost 截断到 `C` 之后，平均值最多 `theta`。

cost 超过 `C` 的 token 按 `C` 计费，所以它可以是任何 token。这样的 token 最多 `cap` 个。接受时打印 `WITHIN LIMITS: …`。

这个 cost 基于 Token-DiFR 在温度 0 下的分数（arXiv 2511.20621），有四个差异（[PUBLISHED-WORKS.md:58-66](../PUBLISHED-WORKS.md)）：平局计 1；每个位置加已知偏移 $2^{\mathrm{sh}}$；对截断的位置数有上界；阈值 `theta` 是参数文件里的一行，由做承诺的一方设定，程序不校准它。

commitment ID 不能被读出是哪种策略，所以 client 要用 `-policy clipped-gap,C,THETA,CAP,S,SH` 说明它期望的数字，每个值必须等于 `policy.txt` 的对应行（[guide/how-it-works.md:69-70](../guide/how-it-works.md)；[verifier/README.md:110-113](../verifier/README.md)）。

### 5.4 `WITHIN LIMITS` 说了什么，没说什么

`WITHIN LIMITS` 只说：被检查 token 的 cost 合起来满足 client 命名的限制。它不说回复是模型自己的。限制宽松或 logit 接近时，几乎任何 token 都能通过，verifier 看不出来（[guide/how-it-works.md:72-74](../guide/how-it-works.md)）。

两个数字（都标 †，不是在发布的程序上）：两个常见推理引擎各 96 条诚实回复，没有一条能通过 exact match；32 条「把一个 token 换成模型第二选择」的回复里，28 条满足按诚实回复拟合的限制。项目不推荐任何限制：如果你不能评估它们，只接受 exact match（[guide/known-limits.md:45-49](../guide/known-limits.md)）。

没有任何运行显示 clipped gap 的限制在 verifier 处拒绝过什么：没达到限制的回复从诚实 prover 那里得不到证明（[guide/known-limits.md:43-45](../guide/known-limits.md)）。

Lean 对 clipped gap 只证明了「被接受的输出满足该策略」。这对一个对权重作承诺、而该权重下所有输出都满足策略的 committer 不构成任何阻碍（[docs/protocol.md:55](../docs/protocol.md)）。

`result-v1` 里的 `above-clip=` 说限制允许多少个 token 的 cost 超过 `C`。exact match 下是 0（[verifier/docs/output.md](../verifier/docs/output.md)，「The field `above-clip=`」）。

### 5.5 `YES` 说了什么，没说什么

`YES` 和 `WITHIN LIMITS` 的退出码都是 0，所以要看最后一行，不能只看退出码（[verifier/README.md:146-149](../verifier/README.md)）。只检查 $y$ 的调用也打印 `YES`。

一个 `YES` 覆盖的只有被检查的 token。截短的回复按更短的回复通过。要自己构造请求文件，把 `covered:` 行和自己的范围比较，检查回复以 end token 结尾。一个挂起或被杀的调用按未接受处理（[guide/known-limits.md:51-56](../guide/known-limits.md)）。

`YES` 不说 closeness round 依赖哪个来源，也不说是否计入了 proof of work（除非你传了 `-no-proof-of-work`）。没有 verifier 检查 provider 是否 mask 了证明或公开文件（[verifier/README.md:259-263](../verifier/README.md)）。

打印的 bit 数不是安全等级。最弱的一轮打印 128.00 bit，其中最多 32 bit 是按假设计入的 proof of work（不计是 96.23），没有余量。这个 accounting 依赖两个被引用而未被重新推导的已发表结果（一个 proximity test 的可靠性分析和 theorem on codes），加上一份不公开的技术报告里的纸面论证。证明了的数字是 Lean 定理的：每次哈希查询约 124 bit，下限约 $2^{-101}$。两个已发表的对 Fiat-Shamir 变换的攻击（ePrint 2025/118；2026/1838）没有针对这个协议或这些程序检查过（[guide/known-limits.md:65-76](../guide/known-limits.md)）。

---

## 6. 权重隐藏

### 6.1 目标和声明清单

weight hiding 是一个目标：client 和公众从 provider 发布和发送的东西里，不能了解任何关于权重的事，除了一份声明清单（[guide/weight-hiding.md:7](../guide/weight-hiding.md)）。清单上至少 11 样可见（[docs/what-is-not-hidden.md:9-21](../docs/what-is-not-hidden.md)）：每条回复；一个请求是被服务还是失败；时间；架构和所有尺寸；从权重拟合的设置（尺度）；`cp_commit` 的检查通过了；一个 batch 失败或服务停止；provider 服务了多少、何时；谁和谁同一个 batch、有几个；长度、batch 里的位置和 nonce；权重的 masked sums（就是 OOD answers）。

清单说的是按设计可见的东西，不是说其他都隐藏了。没有人证明清单完整，也没有人评估这些项合起来泄露什么（[docs/what-is-not-hidden.md:23](../docs/what-is-not-hidden.md)）。

### 6.2 masking 做什么

按设计，清单之外每个依赖权重的公开值要么被 mask（加一个随机值），要么只以带秘密 salt 的哈希发布。masking 默认开启（[guide/weight-hiding.md:7](../guide/weight-hiding.md)）。设计的保护方式（[docs/protocol.md:273](../docs/protocol.md)）：

- 权重表打开的列，在数量不超过 query bound 时看起来随机；程序计数以便在限制前停下。
- 其他被承诺的东西，对被读的位置带足够的自身随机性。
- 只被哈希的东西由 salt 保护。
- 其他发送的值被 mask，除了下一个 token，它明文发送。

一个证明有三部分，各有自己的 prover：key file 的证明（编码后的权重在范围内，做一次）、link proof（另一个编码持有同样的权重）、请求的证明（回复是模型的）（[guide/weight-hiding.md:11](../guide/weight-hiding.md)）。

程序用 ChaCha20（一种流密码）从秘密 seed 和系统的新鲜随机字节派生随机值。所以程序的隐藏至多是 computational（计算意义上的）。技术报告关于两个派生值重复的概率的界，只对满足印出条件并具有某个性质的派生方式成立，两者都没有对发布的程序显示；按字面读，程序的方式不满足其中一些条件（[docs/protocol.md:280](../docs/protocol.md)）。

### 6.3 zero knowledge：证明了什么

**zero knowledge（零知识）** 是 weight hiding 的密码学形式：verifier 看到的一切都可以由一个 simulator（模拟器）在没有权重的情况下，从公开设置和每个证明的 prompt、输出 token 生成（[guide/terms.md:160](../guide/terms.md)）。设置里包含从权重选出的尺度，所以零知识只说权重隐藏在「符合这一个配置的权重」之中（[guide/weight-hiding.md:13](../guide/weight-hiding.md)）。

Lean 证明了的（[guide/weight-hiding.md:15-21](../guide/weight-hiding.md)）：

| 部分 | Lean 有什么 |
|---|---|
| 哈希之前的交互协议（IOP）：三个证明 | 证明了，在完整规模：三个诚实 prover（Lean 定义）的 honest-verifier zero knowledge。每次运行被精确模拟，除非 prover 因私密原因停止：概率 0、0 和至多 $2^{-174}$。一个承诺的整个寿命内（最多 710,324 张 leaf table，每张 688,127 个请求）：$2^{-135}$ 之内 |
| 哈希之后（Fiat-Shamir），random-oracle model | 证明了：编译后的 committer 和 prover，配合被证明可靠的 Lean verifier，对任何在 $2^{64}$ 次哈希查询和 688,127 次回复之内的 verifier，统计距离低于 $2^{-154}$；一个承诺一张 leaf table，每个证明一个请求，exact match，新鲜随机 mask |
| 泄露 | 每个请求一个 bit：served 或 refused；它依赖权重 |
| batch；第二个承诺；时间；公开文件自身透露的 | 没有 |
| C++ 和 CUDA 程序 | 没有：未证明；测试了 |

这些数字是统计距离，不是安全等级。simulator 的运行时间没有建模。定理假设 verifier 在承诺之前不做哈希查询，权重和公开文件在哈希函数抽取之前固定。第二个固定哈希函数不被当作随机（[guide/weight-hiding.md:23](../guide/weight-hiding.md)）。

那一个 bit 的后果：对一个 prompt 逐个尝试候选 token 的人，仅从这个 bit 就能学到模型的下一个 token，除非全部被拒绝。没有任何陈述给出很多 bit 合起来泄露多少的界（[guide/terms.md:161](../guide/terms.md)）。

Lean 的定理是关于一个 Lean 定义的 prover，用新鲜随机 mask。程序从秘密 seed 派生 mask，对程序没有任何证明。batch 是程序实际服务的东西，定理不覆盖（[guide/weight-hiding.md:5](../guide/weight-hiding.md)）。

整个协议的隐藏性的书面证明在技术报告里，不公开也不稳定：检查得太少，几乎每次整读一部分都发现措辞为假的句子。只有一步（权重表上的 OOD samples）被在纸面上有意攻击过（[guide/weight-hiding.md:25](../guide/weight-hiding.md)）。

### 6.4 程序：只测试了

五个测试检查 mask 是否存在、丢失的 mask 是否被发现；没有一个搜寻泄露。项目的判断是：在默认设置、诚实运行、provider 守住它的职责时，weight hiding 很可能成立。这个判断基于设计、Lean 对协议的证明、代码阅读（整文件，读一遍，快速，没有构建或运行）和测试；它不是证明（[guide/weight-hiding.md:29](../guide/weight-hiding.md)）。五个测试里只有第一个和第五个在发布的程序上跑过，只有第一个在真实模型的权重上跑过（[guide/known-limits.md:120-124](../guide/known-limits.md)）。

**坏结果**（[guide/weight-hiding.md:33](../guide/weight-hiding.md)）：一个被 bug 或硬件故障丢掉的 mask，可以留下一个能通过验证的未 mask 值。在 CPU 内存里故意销毁的 165 个随机值中，123 个被拦下，12 个没改变任何文件，30 个没被注意到；GPU 内存里的 130 个全部被拦下（在其他来源的 test build 上，小规模，合成权重，不是比率）。

由此来的几条职责（[guide/weight-hiding.md:38-42](../guide/weight-hiding.md)）：

- **counter**：`isk` 里有 seed 和 counter。复制或恢复的目录有旧的计数；超过限制后证明给出权重的精确方程，无人察觉。永远不要复制、备份或恢复它。
- **seed**：公开文件永久保存权重的 masked sums，谁读到 seed 谁就能去掉 mask。`-public-weights` 下什么都不隐藏。
- **检查**：prover 不验证自己的证明。provider 的 front end 必须在一个 batch 的任何回应离开之前，对该 batch 的每个证明跑 `cp_verify`。一个被 `cp_verify` 拒绝的回应可能含有未 mask 的值。这个检查看不出一个通过验证的证明里丢失的 mask。
- **clipped gap**：每个结果都告诉 client 关于模型分数的一些信息；很多结果合起来告诉多少，没有界。Lean 的零知识定理只对 exact match。

还有一点要注意：3b 的 `cp_verify` 日志里有两行 `weight hiding: NOT CHECKED BY THIS CALL (it rests on how the commitment was made)`（[cp_verify.log:150,208](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)）。这是每个被接受的调用都会打印的固定行。

---

## 7. Lean 里证明了什么，没证明什么

FlashProve 有两个 verifier。C++ 的 `cp_verify` 是用户运行的程序。Lean 写的 `verify` 是为了让定理能被机器检查（[guide/what-is-proved.md:40-42](../guide/what-is-proved.md)）。

### 7.1 证明了的

**Soundness（可靠性）**，对 Lean 的 `verify`：它接受一个假声明的证明的概率，在 $q$ 次哈希查询后至多约 $q/2^{124} + 2^{-101}$，在 random-oracle model 下（把哈希当作只能查询的随机函数），对 exact match 声明（[guide/what-is-proved.md:46-51](../guide/what-is-proved.md)）。条件（[guide/what-is-proved.md:53-68](../guide/what-is-proved.md)）：

- exact match；其他策略只有「被接受的输出满足该策略」。
- random-oracle model，第二个固定哈希函数假设独立，经典对手。
- 三个配置文件在哈希函数抽取之前固定。
- 只对完整规模的 layout 和 proof format，参数文件八个数在给定范围内自由。小示例不在内。
- 一个证明。一次交上来的 $N$ 个证明代价是 $N$ 倍误差；分开会话检查没有陈述的界。

「假」的意思是：在一个固定的 extractor（提取器，一个未显示可计算的数学函数）从承诺和 committer 的哈希查询中读出的权重上为假。这个界是关于假接受的概率，不是关于「一个给定的接受为假」的概率，而且随查询数增长，从约 $2^{124}$ 次查询起什么也不说（[guide/what-is-proved.md:46-51](../guide/what-is-proved.md)）。

三个进一步的定理把每一轮的误差界定为 $1/2^{128}$，它们界定的是单轮，不是伪造的概率。proof of work 被计入：没有它的 20 到 32 bit，抽样位置的轮的误差约 $2^{-96}$ 到 $2^{-110}$（一个程序加总的，没有定理）。124 bit 和 128 bit 这两个数字都不对 C++ 程序声称（[guide/what-is-proved.md:70](../guide/what-is-proved.md)）。

`verify` 的 executable form（可执行形式）计算 `verify` 自己的判决，被证明同样可靠。命令 `lean-verify` 在 prover 的证明文件上运行它（[guide/what-is-proved.md:72-77](../guide/what-is-proved.md)）。

Lean 的证明不假设任何关于编码的猜想：需要的编码理论在库内证明，文件改编自其他形式化，它们和已发表陈述的一致性没有独立比较（[guide/what-is-proved.md:79-80](../guide/what-is-proved.md)）。

**Completeness（完备性）**：诚实 prover 对真声明的证明被接受。对协议证明了，在 served request（能被服务且声明的输出是模型自己的结果的请求）上，exact match，先对交互协议，再对编译后的 prover 和 verifier，拒绝概率至多 $2^{-40}$。模型在请求上没有结果时它什么也不说。它关于 Lean 定义的 prover，不是 CUDA prover（[guide/what-is-proved.md:149-173](../guide/what-is-proved.md)）。

**Zero knowledge**：见第 6.3 节。

### 7.2 没证明的

- C++ verifier、CUDA prover 和 committer：没有任何定理。它们被写成遵循同一个协议，并对照它测试。它们在列出的点上和 Lean 描述的协议不同（[lean/GAPS.md](../lean/GAPS.md)）；这些差异是否重要，两个方向都没有显示（[guide/what-is-proved.md:242-245](../guide/what-is-proved.md)）。
- 编译器和硬件被信任；BLAKE2b 的实现和 ChaCha20 是普通代码，对照已发表向量测试；GPU kernel 部分测试，大多在小规模（[guide/what-is-proved.md:274-277](../guide/what-is-proved.md)）。
- 没有独立的安全审计。AI 系统在人的指导下写了几乎全部软件、Lean 证明和文档，并跑了报告的测试（[guide/known-limits.md:131-133](../guide/known-limits.md)）。
- `blocks/` 库可以被误用：直接调用它、不守 `blocks/chain/closed.h` 契约的程序可以被弄成接受假声明，至少一种方法在 toy 规模上可行。`cp_verify` 没有暴露这一点，依据是读源码，不是测试（[guide/known-limits.md:165-169](../guide/known-limits.md)）。
- 被改动的存储表可以通过验证（观察到最多 8,388,608 个位置中的 131,072 个被改）；每个这样的表仍远在只能译码到诚实权重的距离之内，所以被接受的声明是诚实运行的声明：一个 `YES` 为声明担保，不为存储的每个字节担保（[guide/known-limits.md:171-174](../guide/known-limits.md)）。
- 证明是 malleable（可延展的）：诚实证明的一些字节可以改动而仍被接受。一个接受不说明这个文件正是 prover 发出的那个（[guide/known-limits.md:176-184](../guide/known-limits.md)）。

### 7.3 测试到了什么

在发布的程序上：`examples/maverick/run.sh` 在一个真实模型的公开权重上完整跑过一次，`cp_verify` 以 `YES` 结束；timed runs 里 1,851 个证明全部被接受，249 个改动过的副本和调用全部被拒绝；57 个说谎的证明被拒绝（完整规模，合成权重，一个承诺）；228 个小规模的、在改动过的存储表上的调用被接受（见上一节）。两个 verifier 在共同判断过的每个证明文件上一致（[guide/what-is-proved.md:245-272](../guide/what-is-proved.md)）。

说谎证明的测试很窄：一个承诺、合成权重、exact match、几种谎言、全部在同一个 verifier 步骤被拒绝。没有 prover 被造出来搜索伪造。制造说谎证明的测试程序不在仓库里（[guide/known-limits.md:135-155](../guide/known-limits.md)）。

---

## 8. 这些步骤在你的 3b 日志里

日志在 [repro/2026-11-21-vast-h100/logs/stage3b/](../repro/2026-11-21-vast-h100/logs/stage3b/)。四个文件：`full-f2.out`（脚本输出）、`cp_commit.log`、`cp_prove.log`、`cp_verify.log`。配置是 `target_full_f2`：完整规模，合成权重（`-weights made-up-blocks`），exact match，1 张 leaf table（`-M 1`）（[full-f2.out:4-6](../repro/2026-11-21-vast-h100/logs/stage3b/full-f2.out)）。

### 8.1 协议步骤和日志行的对应

| 协议步骤 | 日志里的证据 |
|---|---|
| 配置的三个文件 | `full-f2.out:3-4`：复制 `policy.txt`、`KL.bin`、`layout.txt` |
| 3.2 编码权重表、建 Merkle 树（anchor） | `cp_commit.log:2`：`W has 2^18 columns, 3076096 rows`；`:128-146`：32 份，`the table's tree: 32 parts: 962.8 s`；`:149`：`phase anchor-table: 977.8 s` |
| 3.5 anchor 的 OOD answers | `cp_commit.log:147-148`：`a walk for the record's answers: 14.7 s` |
| 2.7 worst-case check | `cp_commit.log:117-126`：按层进度，`phase check: 634.6 s` |
| 3.4 range proof（WF）和一次性测试 | `cp_commit.log:150-162`：`the ranges and the rows' vectors: 358.2 s`，`the test on the table: 54.7 s`，key file 3,293,344 字节，`phase key-file: 414.3 s` |
| `cp_commit` 自己用 `cp_verify` 检查 key file | `cp_commit.log:163-222`：`---- stage key ----` 到 `PART PASSED: key` |
| 3.7 commitment ID | `cp_commit.log:225-227`：anchor 的 `RT`（树根）、`h_card`、`commitment ID: 2f50a1aa…` |
| 3.3 leaf table 0 的编码和 link | `cp_commit.log:228-274`：`phase leaf-0-table: 977.6 s`，`the link: 85.6 s`，entry 2,943,544 字节，`phase leaf-0-entry: 96.4 s` |
| `cp_commit` 检查 leaf file | `cp_commit.log:275-329`：`---- stage leaf ----` 到 `PART PASSED: leaf` |
| 4.2 一个 batch，一个请求，leaf 0 | `cp_prove.log:16`：`batch 0 leaf 0 n 1`；`:1`：`at most 1 requests share one opening` |
| 4.5 哈希链不泄露：固定输出 | `cp_prove.log:13`：`p258 closed: the program reads its secrets now. From here on it prints only the fixed lines…` |
| 1.3 `cp_verify` 的三个阶段 | `cp_verify.log:1,20,39`：`stage key … 1.700 s`，`stage leaf … 1.264 s`，`stage request … 21.228 s` |
| 4.3 segment 和 sub-proof | `cp_verify.log:45-69`：25 个 link：`G0`…`G11`（layer group）、`A0`…`A11`（attention）、`H`（head） |
| 4.3 lookup（forest）和 range check（smallness） | `cp_verify.log:71`：`8 lookup calls in one forest, 2 more in the smallness test's, 9 matrices, 803 claims listed` |
| 4.6–4.7 共享链和第二条链 | `cp_verify.log:147-148`：`behind them (the rows on the table, the place, the shared chain, the second chain) 0.207`；`25 links with what they leave on the table of weights; the chain that the whole batch shares; the chain that ends the request` |
| 4.4 slot | `cp_verify.log:164`：`leaves=64 place=58` |
| 5.1 覆盖范围 | `cp_verify.log:209`：`covered:` |
| 3.7 ID 相符 | `cp_verify.log:205-206`：`commitment ID: 2f50a1aa…`，`id: named by the caller and equal` |
| 4.5 proof of work 和 accounting | `cp_verify.log:170-203`，`:214` |

`cp_prove.log` 第 5 行写出了设置：`setting: field degree 3; requests: fold width 128, 5 repetitions mod q; opening: fold width 168, 5 repetitions mod q; identities up to 2^28 keys; folds inside 2^17; …`。`field degree 3` 是三次扩域。`identities up to 2^28 keys` 对应参数文件的 `lidb` 行，lookup 恒等式最多 $2^{28}$ 个 key（[cp_prove.log:6](../repro/2026-11-21-vast-h100/logs/stage3b/cp_prove.log)）。其他数字的定义本笔记没有查到，不展开。

从 `cp_verify.log` 的 link 列表可以看到 12 个 layer group。48 层除以 12 是每组 4 层。这是从日志推算的，不是文档写明的。

### 8.2 阶段用时

来自 [REPORT.md:159-169](../repro/2026-11-21-vast-h100/REPORT.md) 和日志：

| 部分 | 用时 |
|---|---|
| `cp_commit`：权重进主机内存（锁定 236.9 GB） | 65.9 s |
| `cp_commit`：worst-case check（phase check） | 634.6 s |
| `cp_commit`：anchor 的表和树 | 977.8 s |
| `cp_commit`：key file | 414.3 s |
| `cp_commit`：leaf 0 的表和树 | 977.6 s |
| `cp_commit`：leaf 0 的 link（entry） | 96.4 s |
| `cp_prove`：1 个请求，1,000 个 token | 约 2 分钟（含启动，按文件时间估计） |
| `cp_verify` | 约 24 秒（单线程） |
| 合计 | 约 56 分钟（文档写约 42 分钟） |

承诺比文档慢约 35%。报告推测原因在 GPU（H100 PCIe，SM 时钟只有最高值的约 65%，87 °C），没有办法把两个可能原因分开（[REPORT.md:171-186](../repro/2026-11-21-vast-h100/REPORT.md)）。

### 8.3 结尾四行

一次完整调用以四行结束：`commitment:`、`proof of work:`、`result-v1`、判决（[verifier/docs/output.md](../verifier/docs/output.md)，「The closing lines」）。3b 的四行（[cp_verify.log:213-216](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)）：

```text
commitment: 86 closeness rounds of the one-time files rest on another source than the bound within a third of the distance; the table has a record: the binding of the weights rests on the theorem on codes (ePrint 2025/2055, Theorem 4.6), taken as given
proof of work: up to 32 bits counted in 71 rounds; with it the least round has 128.00 bits, without it 96.23 bits
result-v1 status=0 claim=exact-match id=2f50a1aa14592d44e0bcf27b1daaa5d5e94c703a95637ff90309bd8cf5f8ae45 bits=128 work=32 work-rounds=71 least-round=128.00 least-round-without-work=96.23 commitment-assumed-rounds=86 record=1 r=23 e=1024 T=1024 tokens=24..1023 y=yes policy=exact-match C=256 theta=256 cap=0 s=15 sh=8 sT=4 F=24 above-clip=0 leaf=0 stages=key,leaf,request dw=8 build=93afdaf592393a54ebd17b681d1c03295eb883f0161ff0e6ab872a3b39939595
YES
```

读法：

- **`commitment:`**。**closeness round（接近性轮）**是一个挑战把几行合成一行的轮；它的误差主要是「合成的行接近码而原来的行并不一起接近」的概率（[docs/glossary.md:122-126](../docs/glossary.md)）。86 个这样的轮属于 key file 和 leaf file（63 + 23，见 [cp_verify.log:4,23](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)），它们依赖 theorem on codes 而不是「距离三分之一以内的界」（那是在协议所依据的论文里证明了的引理）。`the table has a record` 说表有 OOD answers。这个计数不决定任何事：没有哪个计数会把 `YES` 变成别的结局（[verifier/docs/output.md](../verifier/docs/output.md)，「The `commitment:` line」）。
- **`proof of work:`**。最多 32 bit；71 轮只有计入 proof of work 才达到 target；计入时最弱的轮 128.00 bit，不计时 96.23 bit。两个可以是不同的轮（[verifier/docs/output.md](../verifier/docs/output.md)，「The line on proof of work」）。
- **`result-v1`**。给程序读的行。`commitment-assumed-rounds=86` 和 `commitment:` 行的 86 是同一个数。`record=1`：表有 OOD answers。`r=23 e=1024 … y=yes`：和 `covered:` 一致。`bits=128`：调用要求每轮至少 128 bit，这是你的要求，不是你得到的安全性（[verifier/README.md:129-133](../verifier/README.md)）。
- **`YES`**：在 exact match 下接受。退出码 0（[full-f2.out:19-22](../repro/2026-11-21-vast-h100/logs/stage3b/full-f2.out)）。

accounting 行里还有两个有用的数字：`THE LARGEST ROUND ERROR: 2^-128.00, in the round "RNG (G0, smallness test)"`，共 67,383 轮，62,421 轮有 verifier 的消息（[cp_verify.log:170](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)）；`341 of them rest on the theorem on codes … TAKEN AS GIVEN`（[cp_verify.log:189](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)）；`as one number, by the compiled bound: 126.21 bits, proof of work counted (without it: 96.23 bits)`（[cp_verify.log:186](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)）。日志自己写着：`that the programs compute the protocol is not proved; no figure here is a security level`（[cp_verify.log:197](../repro/2026-11-21-vast-h100/logs/stage3b/cp_verify.log)）。

### 8.4 这次 `YES` 说明了什么

按 [AGENTS.md](../AGENTS.md) 的报告规则和 [REPORT.md:267](../repro/2026-11-21-vast-h100/REPORT.md)：这次 `YES` 说明，在这台 H100 PCIe 上，这个版本的程序能构建，能对合成权重做承诺、生成证明，`cp_verify` 接受这些证明。权重是合成的，不是 Llama 4 Maverick 的。它不说明任何 provider 的模型，也不说明权重被隐藏。

---

## 9. 为什么论文的机器有 8 块 GPU，而只用了 1 块

这个问题之前没有回答。Drive 上的总览报告 `2026_rinberg_flashprove_library.pdf` 可以读到，答案在它的第 6 节「Implementation: running a 400B model on one GPU」和第 7 节「Limitations」。

- **prover 只用一块 GPU。** 报告说 prover 跑在 provider 一侧，一台有一块 H100 和 24 个 CPU 核的服务器上。模型的权重占 232 GB，留在主机内存（机器的普通 RAM）里。GPU 只有 80 GB 显存，它把权重流式读入，每次只保存 trace 的一个 segment。这就是第 4.3 节 segment 设计的原因：报告第 5.4 节说，简化的 IOP 把整个 trace 作为一个 oracle 发送，prover 要保存整个 trace 和它的编码直到最后的 proximity test，这远超一块 GPU 的容量，而 FlashProve 的设计目标就是在一块 GPU 上运行。
- **8 块 H100 用在对照实验上。** 报告在「Comparison with plain inference」一段说，他们用 vLLM 跑了同一个模型的 FP8 版本，在 8 块 H100 上。对比结果是：按每块 GPU 算，证明一个 24 个请求的 batch 大约比只做 prefill 慢 72 倍（各次运行 66 到 77 倍），比 vLLM 在他们试过的最高负载下的 prefill 慢约 94 倍。也就是说，8 块 GPU 的机器是普通推理的基线所需要的，不是 prover 需要的。
- **多 GPU 证明不支持。** 第 7 节「Limitations」说：prover 在一块 GPU 上运行，权重存在主机内存，生成证明时流式送到 GPU。tensor parallelism（张量并行，把一个矩阵切到多块 GPU 上）和多 GPU 证明不支持。一个多 GPU 的 prover 主要能降低单个请求的延迟，用现有技术可行，留作未来工作。

和仓库文档一致：[guide/how-it-works.md:9-10](../guide/how-it-works.md) 写 `cp_commit` 和 `cp_prove` 在「one NVIDIA H100」上运行。3b 日志里也能看到权重留在主机内存：`236.9 GB of host memory page-locked for 434 tensors`（[cp_commit.log:114](../repro/2026-11-21-vast-h100/logs/stage3b/cp_commit.log)）。

---

## 10. 术语表

| 英文 | 一行中文解释 |
|---|---|
| acceptance policy | 接受策略：被检查 token 必须满足的测试，exact match 或 clipped gap |
| anchor | 锚表：一个承诺的第一张权重表；标准术语 first table |
| batch | 批：`cp_prove` 对同一张 leaf table 一起证明的一组请求 |
| batch tree / binding tree | 一个 batch 里各请求的 seal 组成的带盐 Merkle 树，树根进入共享链 |
| BLAKE2b | 程序用的哈希函数 |
| binding | 绑定：承诺最多固定一组权重；文档区分几种意思，见 3.7 |
| chain | 哈希链：一条 Fiat-Shamir transcript，每个挑战是它的哈希 |
| checked token | 被检查的 token：接受策略作用的 token，由请求的范围决定 |
| clipped gap | 截断差值：把所有被检查 token 的 cost 一起打分的策略，接受时打印 `WITHIN LIMITS` |
| closeness round | 接近性轮：一个挑战把几行合成一行的轮 |
| code switch (CS) | 换码：把折叠后的行在新的码下重新编码成一个新的 oracle message |
| column | 列：编码表所有行在同一位置的值，也是 Merkle 树的一片叶子 |
| commitment | 承诺：`cp_commit` 从权重和配置做出的东西：公开文件、秘密文件和 ID |
| commitment ID | 64 个十六进制数字，标识一个承诺；要从可信方获得 |
| completeness | 完备性：诚实 prover 对真声明的证明被接受 |
| configuration | 配置：`policy.txt`、`KL.bin`、`layout.txt` 三个公开文件 |
| cost | 代价：一个 token 的 score gap 加 $2^{\mathrm{sh}}$ |
| counter (`count.bin`) | 计数器：每张权重表一个文件，每显示一列增长一字节，从不缩短 |
| default setting | 默认设置：仓库里所有示例参数文件的证明参数，计入 proof of work |
| digit / cell / limb | 数位：权重按基 256 拆成的有符号字节，一个 cell 存一个 |
| exact match | 精确匹配：每个被检查 token 必须是整数模型的首选 |
| extractor | 提取器：Lean 可靠性定理里从承诺和哈希查询读出权重的数学函数，未显示可计算 |
| Fiat-Shamir | 用哈希代替 verifier 发挑战，使协议非交互 |
| field (F, E) | 域：$p = 2^{64}-2^{32}+1$ 和它的三次扩域 |
| filler | 填充 token：补满窗口的 token，两边必须相同 |
| fold | 折叠：用随机因子把一组行合成一行 |
| forest | `cp_verify` 里一组 lookup 调用共用的 key 侧乘积树 |
| full size | 完整规模：Llama 4 Maverick 400B 的形状，48 层，128 个 expert，1,024 token 窗口 |
| hiding polynomial | 隐藏多项式：masked sumcheck 加到被加项上的随机多项式 |
| integer model | 整数模型：权重和激活都量化、只用整数计算的网络；每个证明都关于它 |
| IOP / IOPP | 交互式预言机证明 / 邻近性的交互式预言机证明 |
| Johnson radius | Johnson 半径：一个词附近码字个数有限的距离界 |
| key file (`key.bin`) | anchor 的树根、OOD answers 和 committer 的一次性证明 |
| leaf file | leaf table 的树根、OOD answers 和它的 link proof；Lean 叫 entry file |
| leaf table | 叶表：同一组权重的另一张表，有自己的随机元素；证明对它做；标准术语 copy |
| Lean | 一个证明助手，检查用它的语言写的证明 |
| linear claim (`Row`) | 线性声明：某个线性形式作用在被承诺消息上等于某个公开值 |
| link / link proof | 链接证明：leaf table 和 anchor 编码同一组权重的证明 |
| logit | 整数模型在一个位置上给一个 token 的分数 |
| lookup (LKP) | 查表论证：一个被承诺向量的每个元素都出现在公开表里 |
| mask | 掩码：和秘密值合并的随机值；权重表里是每行的随机元素 |
| Merkle tree | 哈希树：叶子是列的哈希，根承诺所有列 |
| next token (`y`) | 窗口之后的 token；只在 $e = T$ 时被检查 |
| no-overflow conditions | 不溢出条件：`cp_verify` 对公开参数的不等式测试，意在让模 $p$ 的等式在整数上也成立 |
| OOD sample / OOD answers (record) | 域外抽样：对被承诺消息在随机点上的取值；权重表的回答随承诺永久公开 |
| oracle message | 一组编码后的行，在一棵 Merkle 树下承诺；verifier 只读几列 |
| pad | 遮盖一个 OOD answer 的随机值 |
| parameter file (`policy.txt`) | 参数文件：模型形状、证明参数和接受策略 |
| prefill | 对 prompt 和回复一起做的一次前向传播；证明关于它 |
| proof of work | 工作量证明：某些轮之前的哈希搜索，让重试更贵；不降低单次挑战的概率 |
| proximity test | 邻近性测试：证明被承诺的行接近满足 linear claim 的码字 |
| query bound | 查询上限：一行的随机元素能掩盖的位置数 |
| random-oracle model | 把哈希当作只能查询的随机函数的理想化 |
| range check (`smallness`, RNG) | 范围检查：每个 segment 里对 trace 值的近似范围证明 |
| range proof (WF) | 范围证明：key file 里对每个权重 digit 的 lookup 证明 |
| Reed-Solomon code | 用多项式求值做的纠错码 |
| request (`.req`) | 请求文件：$T$ 个 token 加 $r$、$e$，由 client 自己构造 |
| round | 轮：prover 的一条消息加随后的挑战 |
| salt | 盐：和列一起哈希的随机字节，防止猜测 |
| score gap | 分数差：一个 token 的 logit 比最大 logit 低多少，平局时加 1 |
| seal | 封印：一条链末尾的摘要，后面的链用它 |
| seed | 种子：`cp_commit` 抽的 32 字节随机数，存在 `isk` 里 |
| segment / sub-proof | 段 / 子证明：一个 layer group、它的 attention 或输出头的证明部分 |
| served request | 能被服务且声明输出是模型自己结果的请求 |
| shared chain / shared vector ($\Omega$) | 共享链：一个 batch 共用的链，证明共享向量 $M \cdot e^*$ |
| slot (`place`) | 请求在 batch tree 里的位置 |
| soundness | 可靠性：假声明只以小概率被接受 |
| sumcheck | 把「很多项的和」的声明归约成「多项式在随机点的值」的声明的协议 |
| synthetic weights | 合成权重：程序自己生成的权重，不读模型文件 |
| target | 目标：每轮误差必须达到的界，$2^{-128}$；不是安全等级 |
| technical report | 技术报告：项目内部的、不公开也未评审的文档，含纸面论证 |
| theorem on codes | 编码定理：ePrint 2025/2055 的 Theorem 4.6，程序的 accounting 把它当作给定 |
| top choice | 首选：一个位置上 logit 最大的 token，平局取 id 最小 |
| trace | 一个 segment 的中间值，prover 对它承诺 |
| weight hiding | 权重隐藏：客户和公众除声明清单外不了解权重 |
| weight table / weight matrix ($M$) | 权重表：全部权重的 digit 排成的矩阵，每行带随机元素编码 |
| window | 窗口：prompt 和回复排成的 $T$ 个 token |
| `WITHIN LIMITS` | clipped gap 下的接受行，比 `YES` 弱 |
| `YES` | exact match 下的接受行；退出码 0；它本身不足以接受 |
| zero knowledge / index privacy | 零知识：verifier 看到的一切都能由不知道权重的 simulator 生成 |
