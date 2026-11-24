# FlashProve 序列长度实验：T = 2,048（lt 11），Vast.ai 单卡 H100（2026-11-24）

状态：**进行中**。GPU 上的运行还没有结果。本页先记录已经确定的事实。

## 结论（到目前为止）

这次实验想回答一个问题：FlashProve 的程序能不能处理 2,048 个 token 的上下文？token 是模型处理文字的基本单位。一个 token 大约是一个词或半个词。仓库自带的完整规模配置 `target_full_f2` 用 1,024 个 token（参数文件里的行 `lt 10`，T = 2^10）。

| 问题 | 答案 |
|---|---|
| 只把 `lt 10` 改成 `lt 11`，程序能跑吗？ | **不能。** 验证器的规划调用（planning call）拒绝这个参数文件：`NO: forest-keys`。`cp_commit` 开始前会做同一个规划调用，所以 GPU 上什么都不会开始。 |
| 原因是什么？ | 输出头（output head）的查表证明里，键（key）的总数到了 $2^{32}$。程序要求少于 $2^{32}$。详细解释见下面「限制 1」。 |
| 改参数能绕过去吗？ | 只改参数不行。键的总数由词表大小 V 和 T 决定。保持 V = 202,048（Llama 4 Maverick 的词表）时，T = 2,048 一定超过。 |
| 改代码能绕过去吗？ | 有两条路。路线 A：放宽代码里的上限到 $2^{32}$，保持 V = 202,048。路线 B：不改这个上限，把 V 减到 131,072，再改 6 行参数。两条路都还要改 `rotary-table` 这个条件（见「限制 4」）。 |
| 已经做了什么？ | 改了 `rotary-table` 条件的代码，并扩展了对应的测试（提交 `78092cbc`）。路线 B 的参数文件已经通过规划调用。路线 A 由另一个会话在实例上尝试。 |

**重要：** 这个分支上的 `cp_verify` 是改过的程序。它在 lt 11 下给出的 `YES`，是改过的 C++ 验证器的 `YES`。Lean 里证明过的验证器没有改，`guide/what-is-proved.md` 里的结论不覆盖它。

## 机器

和 2026-11-21 的复现用的是同一台机器（Vast.ai 机器编号 149333）。Vast.ai 上内存份额 ≥ 280 GB 的 H100 只有这一台。所有 1x H100 SXM 的内存份额只有 109–129 GB，2x SXM 是 258 GB。`run_target_example.sh` 要求至少 280 GB 可用内存。

| 项目 | 值 |
|---|---|
| 实例 | Vast.ai 实例 62262723，可中断（interruptible），2026-11-23 约 23:25 UTC 创建 |
| GPU | NVIDIA H100 PCIe，81,559 MiB 显存，compute capability 9.0，最高 SM 时钟 1,755 MHz，功耗上限 350 W |
| 驱动 | 620.70.06 |
| CPU | AMD EPYC 7713，分到约 36.6 核，有 `sha_ni` |
| 主机内存 | Vast.ai 页面写的份额是 293 GB；容器的内存上限（cgroup `memory.max`）是 780,689,997,824 字节，约 780 GB |
| 系统 | Ubuntu 24.04.5，Linux 7.0.0；g++ 13.3.0；Python 3.12.3 |
| 价格 | 出价 $0.700/hr，加硬盘 $0.044/hr，共 $0.744/hr；流量 $40.96/TB |

CUDA 13.2 Update 2 的安装方法和上次一样：用 pip 把 `cuda-toolkit[...]==13.2.2` 装到 `~/cuda-13.2.86`，`nvcc` 显示 `V13.2.86`，退出码 0。所有程序都用普通账号 `tester` 运行。

## 准备 lt 11 的参数文件

参数文件 `policy.txt` 的行 `lt` 决定上下文长度 T = 2^lt。它还有 32 行表的摘要（digest），`tsilu0` 到 `texp7`。其中 8 行 `trope0..7` 是 rotary 表的摘要。rotary 表存每个位置的旋转角的余弦和正弦。模型用它表示 token 的位置。T 变了，rotary 表就变了，摘要也要重算。

```
sed 's/^lt 10$/lt 11/' prover/examples/target_full_f2/policy.txt > base
python3 -I scales/policy_tables.py base > policy.txt
python3 -I scales/policy_tables.py --check policy.txt   # the 32 digest lines match the tables
```

`KL.bin`（每层的量化比例）直接复制 `target_full_f2` 的。它只有各层的记录，和 T、V 无关。

## 发现的限制

下面按遇到的顺序列出 lt 11 下的每一处限制。前三处由验证器的规划调用报出。规划调用是 `cp_verify -plan ...`。它只读公开的配置文件，不检查证明，在 CPU 上几秒钟跑完。

### 限制 1：`forest-keys`（键的总数到了 $2^{32}$）

三个规划调用对只改了 `lt` 和 `trope*` 的文件都返回 `NO: forest-keys`，退出码 1：

```
cp_verify -plan accounting <folder> 128
cp_verify -plan request policy.txt KL.bin 128 -prompt <2,050 words> -reply <1 word>
cp_verify -plan arrays <folder> 128
```

**这是什么。** 证明里有很多「查表」（lookup）的步骤。一个查表步骤声明：一组值（键，key）每一个都是一张公开表里的某一行。FlashProve 把一个子证明里的所有查表合成一个「森林」（forest）来证明。森林里每张表的每一行有一个计数（multiplicity），记录有多少个键落在这一行。这个计数用 32 个 bit 存。所以一个森林的键总数必须少于 $2^{32}$。代码位置：`verifier/src/vf_lookup_argument.h:130`（`forestShape`，`h.v() >= 32` 时拒绝）和 `prover/hiding/forest_plan.h:102`。底层引擎 `blocks/iop/lkp.h:306-308` 写了同样的条件。

**为什么 lt 11 会超。** 我们从 T = 1,024 的规划输出反推了每个森林的键数（取 log2，记为 v）。方法：`-plan request -fields` 列出证明的每个字段，一个森林的 sumcheck 轮数是 $v(v-1)/2$。

| 子证明 | v（T = 1,024） | v（T = 2,048） |
|---|---|---|
| 注意力（attention）A | 29 | 31 |
| 层（layers）G | 30 | 31 |
| 输出头 H | 31 | **32** |

输出头的森林在 T = 1,024 时已经是 $2^{31}$。它每个查表调用的键数是 $2^{l_J + \mathrm{lt}}$（`verifier/src/cp_verify.cpp`，`c.n = cd.lJ + lt`）。$l_J$ 由词表大小 V 决定：输出头要检查每个 token 的全部 V 个 logit。logit 是模型给每个候选 token 打的分。所以 T 翻倍，键数翻倍，到 $2^{32}$。

参数行 `lidb`（树的深度上限）帮不上忙：深度已经被最小的调用限制了。把 `callH` 行拆开也不减少键的总数。

### 限制 2：`hid-k-range`（矩阵超过 8,191 列）

把 V 减半以后，规划调用报 `NO: hid-k-range`。原因：几个承诺的矩阵的列数超过上限 8,191（`verifier/src/vf_smallness_packing.h`，`KCAP`）。每个矩阵有 $2^{\mathrm{lm}}$ 行，列数等于单元数除以行数。单元数随 T 增长，行数不变，列数就超了。

从 T = 1,024 的 `-plan arrays` 输出看最宽的矩阵：

| 矩阵 | T = 1,024 的列数 | 随 T 的增长 | 对应的参数行 | 改法 |
|---|---|---|---|---|
| `as`（注意力的分数）| 7,200 | 4 倍（单元数含 $2^{2\,\mathrm{lt}}$）| `lmS 16` | 改成 `lmS 18` |
| `af`（前馈层的激活，3 份）| 5,990 | 2 倍 | `lmaf 16` | 改成 `lmaf 17` |
| `hf`（输出头的 logit 边距等，4 份）| 4,816 | 2 倍 | `lmhf 16` | 改成 `lmhf 17` |

### 限制 3：`policy-iotaA` 和 `shape-call-H-*`（派生值要重写）

行 `iotaA_0` 必须等于 32 加注意力森林的树深度 dT。lt 11 时 dT 从 25 变成 26，所以 `iotaA_0 57` 要改成 `58`。行 `iotaH_0..5` 必须等于输出头每个调用的键位数减 dT。V = 131,072 时 `iotaH_0` 和 `iotaH_2` 从 1 变成 2。这些行「没有自由度」：程序自己算出一个值，文件里必须写同一个值。

### 限制 4：`rotary-table`（served condition，lt ≤ 10）

验证器只在「标准参数形式」下给 `YES`。它是一组按名字检查的条件（`verifier/docs/parameter-file.md`）。其中 `rotary-table` 要求 `lt` 最多 10。原因：仓库的测试 `scales/tests/table_ties.py` 只检查到 1,024 个位置。这个测试看 rotary 表的每一项离「舍入边界」（k + 1/2）有多远。离得太近，不同平台的数学库就可能算出不同的表。

`cp_commit` 开始前会运行旁边的 `cp_verify -plan request`。规划调用说文件不是标准形式时，`cp_commit` 拒绝这个文件（`cp-commit-not-standard`）。所以不改这个条件，GPU 上什么都跑不起来。

### 其他在 lt 11 下不构成限制的条件

| 条件 | 要求 | lt 11 时的值 |
|---|---|---|
| softmax 表（prover，`parameter_file.h`）| lt + S + 1 ≤ 24，S = 12 | 24，刚好满足 |
| prover 的注意力（`prover/attn/fork.cuh`）| lt ≤ 14 | 11 |
| `la`（`la 16`）| la ≥ lt + 1 | 16 ≥ 12 |
| `lB`（`lB 6`）| lB ≤ lt − 2 | 6 ≤ 9 |
| `lmS` | lmS ≤ log2 G + lt + lB = 20 | 18 |

## 代码改动（提交 `78092cbc`）

只改了 `rotary-table` 条件。改动有三个文件：

1. `verifier/src/vf_common.h`，函数 `rotaryServed()`：theta0 = 500,000 时允许 lt ≤ 11；theta0 = 10,000 时仍然只到 lt ≤ 10；dh ≤ 128 不变。theta0 是 rotary 表的底数。`target_full_f2` 用 500,000。
2. `scales/tests/table_ties.py`：theta0 = 500,000 时检查 2,048 个位置（原来 1,024）。theta0 = 10,000 时仍然检查 1,024 个位置。下限（floor）不变。
3. `verifier/docs/parameter-file.md`：`rotary-table` 那一行和「lt 最多 10」那一句。

**测试结果。** 完整的 `table_ties.py` 在 Mac 上运行，输出 `OK: 26 checks`，退出码 0。theta0 = 500,000、2,048 个位置：262,144 项，离舍入边界最近 9.321e-08（下限 9.3e-08），数学库至少要偏 2,559 个单位才会改变一项（下限 2,500）。这两个最小值出现在位置 387，也就是在 1,024 以内，和原来的测试一样。另外单独看位置 1,024–2,047：theta0 = 500,000 最小距离 3.64e-06，最小余量 7,300 个单位，满足下限；theta0 = 10,000 最小距离 9.01e-06，余量 73,600 个单位，**低于**它的下限 100,000。所以 theta0 = 10,000 不放宽。日志：[logs/ties/](logs/ties/)。

没有改 `forest-keys` 的上限（路线 A 的改动在另一个会话，见下面）。

## 路线 B 的参数文件（V = 131,072）

文件在 [kit-lt11-V131072/](kit-lt11-V131072/)。和 `target_full_f2/policy.txt` 相比，改了这些行：

| 行 | 原值 | 新值 | 原因 |
|---|---|---|---|
| `lt` | 10 | 11 | T = 2,048 |
| `V` | 202048 | 131072 | 让输出头的森林回到 $2^{31}$ 个键（限制 1）。V ≤ $2^{17}$ 时，调用 `R16b` 只占一棵树，`FEa`、`FEb` 各不超过 $2^{16}$ 个键；森林共 13 棵树，dT 27，v = 31 |
| `lmaf` | 16 | 17 | 限制 2 |
| `lmS` | 16 | 18 | 限制 2 |
| `lmhf` | 16 | 17 | 限制 2 |
| `iotaA_0` | 57 | 58 | 限制 3 |
| `iotaH_0`，`iotaH_2` | 1 | 2 | 限制 3 |
| `trope0..7` | — | 重算 | rotary 表变长 |

`group` 和 `stages` 行（安全计划）没有改。

**规划调用的结果（用发布版的 `cp_verify`）：**

- `-plan request`（带布局文件）：退出码 4，`PLAN ONLY: no proof was checked`。布局文件合法。每个层子证明 275 行（T = 1,024 时 149），注意力子证明 176 行（56），输出头 175 行（127）。规划器还打印 `plan: not the standard parameter form (rotary-table)`：这是限制 4，改过的 `cp_verify` 不应再打印它。
- `-plan accounting`：proof 最多 24,841,412 字节，平均 24,362,308（T = 1,024：最多 19,254,524，平均 18,775,491）；key 和 entry 的大小不变（3,325,152 和 2,955,040）；一份存储的表能服务 688,127 批，不变。
- 布局文件：`cp_verify -plan arrays` 加 `pack_layout.py --arrays ... 22 4`，输出 `5543 rows over F, 25 rows over E, 5954 blocks`。

注意：V = 131,072 不是 Llama 4 Maverick 的词表大小（202,048）。路线 B 的模型形状和 Maverick 只差这一点。

## 路线 A（另一个会话在做）

另一个会话在同一台实例上尝试路线 A：把 `forest-keys` 的上限从「少于 $2^{32}$」放宽到「最多 $2^{32}$」（`vf_lookup_argument.h` 和 `forest_plan.h` 各一行），保持 V = 202,048。要注意两点：

- 底层引擎 `blocks/iop/lkp.h` 的 `lkp_fault()` 写明条件是 $2^{v} < 2^{\text{planes}}$（planes = 32）。`cp_prove` 和 `cp_verify` 运行时不调用 `lkp_fault()`（只有测试调用），所以程序可能跑得通。但这偏离了引擎写明的条件。
- 一个森林有 $2^{32}$ 个键时，键的编号刚好用满 32 位。代码里如果有 32 位的编号或移位，可能溢出。这要靠运行来发现。

它的结果出来以后会补进本页。

## 费用

实例从 2026-11-23 约 23:25 UTC 开始计费，$0.744–0.779/hr。最终费用在销毁实例以后补上。
