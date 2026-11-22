# FlashProve 复现记录：Vast.ai 单卡 H100（2026-11-21）

状态：**进行中**。这份记录会随着每一步更新。

## 结论（到目前为止）

| 步骤 | 状态 | 结果 |
|---|---|---|
| 1. 检查机器 | ✅ 完成 | 满足全部要求（见下面「机器」） |
| 2. 克隆仓库 | ✅ 完成 | `f7ed/FlashProve`，提交 `9e2bdd8f`，版本 0.16.0 |
| 3. 安装 CUDA 13.2.86 | ✅ 完成 | `nvcc` 显示 `V13.2.86`，退出码 0 |
| 4. 路线 1：构建验证器，检查仓库自带的证明 | ✅ 完成 | 最后一行 `YES`，退出码 0 |
| 5. 阶段 3a：`examples/quickstart-gpu.sh`（8 层合成模型） | ✅ 完成 | `quickstart-gpu: all steps ended as expected`，退出码 0；`cp_verify` 最后一行 `YES` |
| 6. 阶段 3b：完整规模，合成权重 | ⏳ 运行中（2026-11-22 03:15 UTC 开始） | — |

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

## 这些结果能说明什么

- 路线 1 的 `YES` 只说明：`cp_verify` 能在这台机器上构建和运行，仓库自带的示例文件（小模型，合成权重）彼此一致。它不说明任何提供方的真实模型。
- 这些位数（bits）不是安全等级（见 `guide/known-limits.md`）。
- `YES` 是 C++ 验证器 `cp_verify` 给出的。在 Lean 里证明过的是 Lean 版验证器，针对 exact-match 声明。C++ 验证器和 CUDA prover 按同一个协议写成，并经过测试，但它们本身没有被证明（见 `guide/what-is-proved.md`）。
- 运行结果不说明权重被隐藏了（见 `guide/weight-hiding.md`）。
