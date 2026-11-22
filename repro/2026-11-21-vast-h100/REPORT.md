# FlashProve 复现记录：Vast.ai 单卡 H100（2026-11-21）

状态：**完成**。路线 1、阶段 3a、阶段 3b 都得到 `YES`，退出码 0。

## 结论

| 步骤 | 状态 | 结果 |
|---|---|---|
| 1. 检查机器 | ✅ 完成 | 满足全部要求（见下面「机器」） |
| 2. 克隆仓库 | ✅ 完成 | `f7ed/FlashProve`，提交 `9e2bdd8f`，版本 0.16.0 |
| 3. 安装 CUDA 13.2.86 | ✅ 完成 | `nvcc` 显示 `V13.2.86`，退出码 0 |
| 4. 路线 1：构建验证器，检查仓库自带的证明 | ✅ 完成 | 最后一行 `YES`，退出码 0 |
| 5. 阶段 3a：`examples/quickstart-gpu.sh`（8 层合成模型） | ✅ 完成 | `quickstart-gpu: all steps ended as expected`，退出码 0；`cp_verify` 最后一行 `YES` |
| 6. 阶段 3b：完整规模（Llama 4 Maverick 的形状），合成权重，1,000 个 token | ✅ 完成 | `status 0`，`proof of work:` 行，最后一行 `YES`；约 56 分钟（文档写约 42 分钟） |
| 7. 按构建提示改权限，再构建一次验证器 | ✅ 完成 | 提示不再出现 |

## 机器

在 Vast.ai 上租的可中断（interruptible）实例。

| 项目 | 值 |
|---|---|
| GPU | NVIDIA H100 PCIe，81559 MiB 显存，compute capability 9.0 |
| 驱动 | 620.70.06（支持 CUDA 13.5） |
| CPU | AMD EPYC 7713，分到约 36 核，有 `sha_ni` |
| 系统 | Ubuntu 24.04.5，Linux 7.0.0 |
| 编译器 | g++ 13.3.0；nvcc 13.2.86（用 pip 安装，方法见 `prover/docs/build.md`） |
| 主机内存 | Vast.ai 页面写的份额是 293 GB；容器的内存上限（cgroup）约 780.7 GB |
| 硬盘 | 50 GB 容器磁盘 |
| 镜像 | `vastai/base-image:cuda-13.2.1-auto`（Ubuntu 24.04） |
| 价格 | 约 $0.778/hr（出价 + 硬盘费），流量另算 |

完整输出：[logs/machine.txt](logs/machine.txt)。

所有程序都用普通账号 `tester` 运行，不用 root。

## 步骤 3：安装 CUDA

按 `prover/docs/build.md` 的「One way to get that compiler」一节，用 pip 把 `cuda-toolkit==13.2.2` 装到 `~/cuda-13.2.86`。

结果：`Cuda compilation tools, release 13.2, V13.2.86`，退出码 0。日志：[logs/install-cuda.log](logs/install-cuda.log)。

## 步骤 4：路线 1

命令：

```
verifier/build.sh
bin/cp_verify examples/accepted-proof/config examples/accepted-proof/ivk examples/accepted-proof/example.req examples/accepted-proof/example.y examples/accepted-proof/example.proof -id "$(cat examples/accepted-proof/ID.txt)"
```

构建：打印 `THE VERIFIER IS BUILT: /home/tester/FlashProve/bin/cp_verify`，退出码 0。日志：[logs/verifier-build.log](logs/verifier-build.log)。

构建时打印了一行提示（`build.sh: note:`），原文如下：

> build.sh: note: another account may be able to change /home/tester/FlashProve (its group or others can write in it, and it has no sticky bit), under which this checkout lies, and so could change what is built, and this script itself; for a build that no other account can touch, keep the checkout below directories that only you can write in; if /home/tester/FlashProve is yours, close it with chmod go-w

意思是：这个目录的权限允许别的账号写入。我们没有按提示改权限，因为 AGENTS.md 要求先得到同意。这台实例上只有我们自己的账号。

检查：

| 项目 | 值 |
|---|---|
| 退出码 | 0 |
| 最后一行 | `YES` |
| 传入的 commitment ID | `79a5a2bc7a469b40df73dedf40abeb740671e8ae8077738bd3743897aeb22183`（来自 `examples/accepted-proof/ID.txt`，仓库自带的示例） |
| `covered:` 里的 `r=` 和 `e=` | `r=60 e=64` |

`commitment:`、`proof of work:` 和 `result-v1` 开头的几行：

```
commitment: 49 closeness rounds of the one-time files rest on another source than the bound within a third of the distance; the table has a record: the binding of the weights rests on the theorem on codes (ePrint 2025/2055, Theorem 4.6), taken as given
proof of work: up to 32 bits counted in 38 rounds; with it the least round has 128.00 bits, without it 96.23 bits
result-v1 status=0 claim=exact-match id=79a5a2bc7a469b40df73dedf40abeb740671e8ae8077738bd3743897aeb22183 bits=128 work=32 work-rounds=38 least-round=128.00 least-round-without-work=96.23 commitment-assumed-rounds=49 record=1 r=60 e=64 T=64 tokens=61..63 y=yes policy=exact-match C=256 theta=256 cap=0 s=16 sh=8 sT=4 F=24 above-clip=0 leaf=0 stages=key,leaf,request dw=8 build=93afdaf592393a54ebd17b681d1c03295eb883f0161ff0e6ab872a3b39939595
```

完整输出：[logs/route1-verify.log](logs/route1-verify.log)。

## 步骤 5：阶段 3a（小模型）

命令（`PATH` 最前面是 pip 装的 nvcc 13.2.86）：

```
examples/quickstart-gpu.sh
```

脚本依次检查机器（GPU、nvcc、g++、库、`sha_ni`、内存、硬盘），构建 `cp_verify`、`cp_commit` 和 `cp_prove`，然后对一个 8 层的合成模型做承诺、证明一个 64 token 的请求，再用 `cp_verify` 验证。

| 项目 | 值 |
|---|---|
| 时间 | 2026-11-22 02:53:56 UTC 开始，03:14 UTC 结束，约 20 分钟（文档写的是 29–34 分钟，在 11 个 CPU 核上测的） |
| 脚本退出码 | 0 |
| 脚本最后一行 | `quickstart-gpu: all steps ended as expected; your files are in /tmp/tmp.XMNd76w4sW` |
| `Compiled by` 行 | 6 行，全部是 `release 13.2, V13.2.86` |
| `cp_commit` 打印的 commitment ID | `ff83bf91ccd2dace06e925f3c33e924be06fb10b7b889c3e8631b48da84b1d75` |
| 传给 `cp_verify` 的 commitment ID | 同上（用的是 `cp_commit` 自己打印的 ID） |
| `cp_verify` 退出码和最后一行 | `status 0`，`YES` |
| `covered:` 里的 `r=` 和 `e=` | `r=15 e=64` |
| 证明的答案 | token 284；key 1093944 字节，entry 536800 字节，proof 4158668 字节 |

`cp_commit` 正常运行完毕。所以这个容器允许 `madvise(MADV_WIPEONFORK)` 这个系统调用。

`commitment:`、`proof of work:` 和 `result-v1` 开头的几行：

```
commitment: 49 closeness rounds of the one-time files rest on another source than the bound within a third of the distance; the table has a record: the binding of the weights rests on the theorem on codes (ePrint 2025/2055, Theorem 4.6), taken as given
proof of work: up to 32 bits counted in 38 rounds; with it the least round has 128.00 bits, without it 96.23 bits
result-v1 status=0 claim=exact-match id=ff83bf91ccd2dace06e925f3c33e924be06fb10b7b889c3e8631b48da84b1d75 bits=128 work=32 work-rounds=38 least-round=128.00 least-round-without-work=96.23 commitment-assumed-rounds=49 record=1 r=15 e=64 T=64 tokens=16..63 y=yes policy=exact-match C=256 theta=256 cap=0 s=16 sh=8 sT=4 F=24 above-clip=0 leaf=0 stages=key,leaf,request dw=8 build=93afdaf592393a54ebd17b681d1c03295eb883f0161ff0e6ab872a3b39939595
```

**和文档不一样的地方：**

- prover 的构建也打印了 `build.sh: note:`，说 `OUT (/home/tester/FlashProve/prover/build)` 所在的目录别的账号可以写入。原因和步骤 4 一样。这个镜像的 umask 是 0002，所以新建的文件和目录对同组用户可写。
- 编译器打印了一些 `warning #550-D: variable ... was set but never used`。文档说 nvcc 13.2.86 会打印这类警告，它们不会让构建失败。

日志（[logs/stage3a/](logs/stage3a/)）：

- `quickstart-gpu.out`：脚本的全部输出
- `quickstart-gpu.log`、`prover-build.log`、`example.log`：脚本自己写的日志
- `cp_commit.log`、`cp_prove.log`、`cp_verify.log`：三个程序的输出

证明文件本身（`example.proof` 等）没有上传。

## 步骤 6：阶段 3b（完整规模）

命令（用阶段 3a 构建好的程序）：

```
W=$(mktemp -d) && WORK="$W/example"
prover/examples/run_target_example.sh target_full_f2 "$WORK"
```

`target_full_f2` 是完整规模的示例配置：Llama 4 Maverick 的形状（48 层），上下文 1,024 个 token，合成权重，exact-match 声明。请求是 24 个 token 的编造提示词，后面是 1,000 个 token，每个都是模型在前面所有 token 之后的首选。

| 项目 | 值 |
|---|---|
| 脚本退出码 | 0 |
| `cp_commit` 打印的 commitment ID | `2f50a1aa14592d44e0bcf27b1daaa5d5e94c703a95637ff90309bd8cf5f8ae45` |
| 传给 `cp_verify` 的 commitment ID | 同上（用的是 `cp_commit` 自己打印的 ID） |
| `cp_verify` 退出码和最后一行 | `status 0`，`YES` |
| `covered:` 里的 `r=` 和 `e=` | `r=23 e=1024`（被检查的 token：`tokens=24..1023`） |
| 证明的答案 | token 52599；key 3293344 字节，entry 2943544 字节，proof 18785436 字节 |
| `cp_verify` 用时 | 三个阶段共约 24 秒（key 1.7 s，leaf 1.3 s，request 21.2 s），单线程 |

`commitment:`、`proof of work:` 和 `result-v1` 开头的几行：

```
commitment: 86 closeness rounds of the one-time files rest on another source than the bound within a third of the distance; the table has a record: the binding of the weights rests on the theorem on codes (ePrint 2025/2055, Theorem 4.6), taken as given
proof of work: up to 32 bits counted in 71 rounds; with it the least round has 128.00 bits, without it 96.23 bits
result-v1 status=0 claim=exact-match id=2f50a1aa14592d44e0bcf27b1daaa5d5e94c703a95637ff90309bd8cf5f8ae45 bits=128 work=32 work-rounds=71 least-round=128.00 least-round-without-work=96.23 commitment-assumed-rounds=86 record=1 r=23 e=1024 T=1024 tokens=24..1023 y=yes policy=exact-match C=256 theta=256 cap=0 s=15 sh=8 sT=4 F=24 above-clip=0 leaf=0 stages=key,leaf,request dw=8 build=93afdaf592393a54ebd17b681d1c03295eb883f0161ff0e6ab872a3b39939595
```

### 用时

| 部分 | 时间（UTC） | 用时 |
|---|---|---|
| `cp_commit`：把权重放进主机内存（锁定 236.9 GB） | 03:15:47 开始 | 65.9 s |
| `cp_commit`：检查权重（phase check） | | 634.6 s |
| `cp_commit`：表的树（32 份，每 2 份约 55 s） | | 977.8 s |
| `cp_commit`：key 文件 | | 414.3 s |
| `cp_commit`：leaf 0（32 份，每 2 份约 55 s） | 03:50:47–04:08:43 | 约 18 分钟 |
| `cp_prove`：证明 1,000 个 token | 04:09:18–04:11:10 | 约 2 分钟 |
| `cp_verify` | 04:11:10–04:12:11 | 约 1 分钟 |
| **合计** | 03:15:47–04:12:11 | **约 56 分钟** |

**比文档慢。** `docs/quickstart.md` 写的是约 42 分钟。几乎所有多出的时间都在 `cp_commit` 里。

我们在 `cp_commit` 运行时看了一次 GPU 状态（03:58 UTC，`nvidia-smi dmon`，5 秒）：

| 项目 | 值 |
|---|---|
| SM 使用率 | 100% |
| SM 时钟 | 1095–1170 MHz（这张卡的最高值是 1755 MHz） |
| 温度 | 87 °C |
| 功耗 | 约 298 W（上限 350 W） |
| PCIe | Gen 4 x16 |

GPU 一直满载，所以这段时间受 GPU 限制。同时 SM 时钟只有最高值的约 65%，温度 87 °C。可能的原因有两个，我们没有办法分开它们：

- 这是 H100 **PCIe** 版。和 SXM 版相比，它的 SM 更少，功耗上限更低（350 W），显存带宽也更低。文档没有写那 42 分钟是在哪种 H100 上测的。
- GPU 可能因为温度降了频。运行结束后再查降频原因时，GPU 已经空闲，所以没有查到运行时的原因。

这只看了一次，结论只是推测。

主机内存：`cp_commit` 在 03:54 UTC 用了约 226.5 GB（RSS）。我们没有记录峰值。

日志（[logs/stage3b/](logs/stage3b/)）：`full-f2.out`（脚本输出）、`cp_commit.log`、`cp_prove.log`、`cp_verify.log`。证明文件本身没有上传。

## 步骤 7：改权限

经过同意，我们按构建提示运行了：

```
chmod go-w /home/tester/FlashProve
```

权限从 `drwxrwxr-x` 变成 `drwxr-xr-x`。只改了这一个目录，没有改里面的文件。阶段 3b 结束以后，我们再运行了一次 `verifier/build.sh`。这次没有 `build.sh: note:` 那一行，退出码 0。日志：[logs/verifier-rebuild-after-chmod.log](logs/verifier-rebuild-after-chmod.log)。

注意：路线 1、阶段 3a 和阶段 3b 用的程序都是在改权限之前构建的。

## 这些结果能说明什么

- 路线 1 的 `YES` 只说明：`cp_verify` 能在这台机器上构建和运行，仓库自带的示例文件（小模型，合成权重）彼此一致。它不说明任何提供方的真实模型。
- 阶段 3a 和 3b 的 `YES` 说明：在这台 Vast.ai 的 H100 PCIe 上，这个版本的程序能构建，能对合成权重做承诺、生成证明，`cp_verify` 接受这些证明。3b 是完整规模的形状，但权重是合成的，不是真的 Llama 4 Maverick（真权重是路线 4，这次没有做）。
- 每个阶段只跑了一次。用时只代表这一次、这一台机器。
- 这些位数（bits）不是安全等级（见 `guide/known-limits.md`）。
- `YES` 是 C++ 验证器 `cp_verify` 给出的。在 Lean 里证明过的是 Lean 版验证器，针对 exact-match 声明。C++ 验证器和 CUDA prover 按同一个协议写成，并经过测试，但它们本身没有被证明（见 `guide/what-is-proved.md`）。
- 运行结果不说明权重被隐藏了（见 `guide/weight-hiding.md`）。
