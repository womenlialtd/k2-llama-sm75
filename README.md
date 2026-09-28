# k2-llama-sm75

**个人用的 Windows + CUDA sm_75 (GTX 1660) 构建仓库。**

这里是 [`ggml-org/llama.cpp`](https://github.com/ggml-org/llama.cpp) 的一个 fork，**不是官方仓库**，也不是一条开发分支。保留上游代码只有一个原因：GitHub Actions 需要一份源码作为编译对象。

源码本身请去上游看：[ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp)

---

## 我在这个 fork 上做了什么

| 分支 | 内容 |
|---|---|
| `master`（默认分支） | 上游代码 + 我加的 2 个文件（构建 workflow 与本 README），**没有把改动提交进上游的 C++ 源码**（构建时会在 CI 里临时打一个补丁，见下节） |
| `model/K2Horizon` | 含 K2-Horizon 模型支持的分支，构建固定在这里的 commit `35999d1` |

提交进仓库的改动，全部加起来就 **2 个文件**（都不碰 C++ 源码）：

```
.github/workflows/k2-horizon-win-cuda.yml   构建 workflow，+393 行
README.md                                   就是本说明（覆盖了 fork 继承来的上游 README）
```

除此之外只有两类事，都不体现在仓库 diff 里：**构建时临时打的 Unicode 补丁**（见下文），以及**编译选项**（只针对 sm_75）。

也就是说：**这个仓库的实际产出都在 Releases 里，不在源码里。**

---

## 下载

Releases → [`k2-horizon-llama-35999d1`](https://github.com/womenlialtd/k2-llama-sm75/releases/tag/k2-horizon-llama-35999d1)

`K2-Horizon-llama-35999d1-bin-win-cuda13.4-x64.zip` （45 文件 / 约 49 MB）

- `sha256: 34392a98adfe699d555f708cb5a71a001f4b9df010dbc6052af0a77960155127`
- 需要已安装 CUDA 13.x 运行时（包内不含 `cublas64_13.dll` 等，与官方分发方式一致）

---

## 这个包跟官方包的区别

| | 官方 `cuda-13.4-x64` 包 | 本包 |
|---|---|---|
| K2-Horizon 模型 | ❌ 不支持 | ✅ 支持（`llama_model_k2_horizon`） |
| sm_75 的机器码 | 只有 PTX，运行期由驱动 JIT | **原生 SASS（`-DCMAKE_CUDA_ARCHITECTURES=75`）** |
| `ggml-cuda.dll` 体积 | 147 MB（多架构 fatbin） | 52 MB（单架构） |

架构结论不是推测：解包 `.nv_fatbin` 的 zstd 帧、按 cubin `e_flags` 的 bits 8–15 解码得到 ——

- 本包 `ggml-cuda.dll`：真 SASS = **`{75}`**（142 个 kernel），PTX target 亦为 `{75}`
- 官方 `llama-b11201-bin-win-cuda-13.4-x64.zip`：真 SASS = `{86, 89, 120, 121}`，sm_75 **只出现在 PTX 段**

原因在上游 `ggml/src/ggml-cuda/CMakeLists.txt` 的默认架构串：`75` 是 `75-virtual`（只出 PTX），而 `86`/`89` 给的是 `-real`。

**代价**：只编了 sm_75，**换非 Turing 显卡会静默回落到 CPU**。CUDA 后端没有其它架构的 kernel。

---

## 构建时打的补丁：`src/unicode.cpp`

这是本仓库除了 workflow 之外实际做的另一件事。**补丁只在 CI 构建时临时应用，不提交进仓库**，所以你在 `master` / `model/K2Horizon` 的 diff 里看不到它。

- **来源**：第三方社区补丁 [`Manus-Vindicte/IssuesFixes`](https://github.com/Manus-Vindicte/IssuesFixes/blob/main/llama.cpp/k2horizon-windows-unicode/k2horizon-windows-unicode-fix.patch)（不是我写的，本仓库只是构建时套用；补丁内部注释也建议它应作为通用修复进上游）。
- **要解决的问题**：MSVC 的 narrow-ECMAScript `std::regex` 不接受 `\uHHHH` 转义，会抛 `error_escape`。K2-Horizon 的分词模板里用了 `\u200C`（ZWNJ 零宽非连接符）/ `\u200D`（ZWJ 零宽连接符），于是这些正则**在这份 Windows 构建上永远加载失败** —— 不报错到用户可见层面，只是分词结果不对。
- **补丁做了什么**：在 `unicode_regex_split()` 里新增一个 `collapsed_byte_for_cpt` lambda（共 2 个 hunk、+66/-1 行），把 pattern 中的每个 `\uHHHH` 改写成"文本折叠逻辑给同一个码点所分配的那个字节"。第二个 hunk 让循环使用改写后的 pattern。
- **为什么这样等价**：它复用的就是同文件里已有的折叠映射函数，不是新发明的行为 —— 各平台匹配结果一致，只是绕开 MSVC 正则引擎的转义限制。
- **应用与校验**（见 workflow 中 `Apply K2-Horizon Windows Unicode patch` 一步）：补丁要求 `src/unicode.cpp` 的 blob 为 `93996f9`；`35999d1` 那份实测 `git hash-object = 93996f9dd542ed2b2e671de51ed98c243d65dabe`，精确匹配。构建时先 `git apply --check`，不干净就**直接 exit 1 让构建失败**（不会静默出一个没打补丁的包），成功后打印 `git diff --numstat` 留证据。那次成功构建（Run 36270572031）打印的 `git diff --numstat` 为 `66 1 src/unicode.cpp`（增 66 行、删 1 行）。

一句话：**本包 = 上游 K2-Horizon 分支 + 一个 Windows 正则兼容性补丁 + 只针对 sm_75 的编译选项。**

---

## 构建参数（复现用）

在 GitHub Actions 里手动触发 `K2-Horizon Windows CUDA 13.4 Build`（`workflow_dispatch`），约 32 分钟。

```
runner        windows-2022
compiler      MSVC (VS2022) + Ninja Multi-Config
CUDA          13.4.1 GA，用 NVIDIA redist 归档免安装部署
source        commit 35999d101cf2233fc54f09c3c8d599da7303ce02（固定）
补丁          k2horizon-windows-unicode-fix.patch（仅 src/unicode.cpp，修 Windows 下的 Unicode 分词）
关键选项      -DGGML_NATIVE=OFF -DGGML_BACKEND_DL=ON -DGGML_CPU_ALL_VARIANTS=ON
              -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=75 -DLLAMA_OPENSSL=OFF
```

两点注意：
- **CUDA 13.4.0 不存在**，实际版本是 13.4.1；`Jimver/cuda-toolkit` action 最高只到 13.3.1，所以这里改成手动下载 redist + `robocopy /E` 摊平。
- 产物目录是 `build\bin\Release\`（Ninja Multi-Config），不是 `build\bin\`。

---

## 相关仓库

同一个用途、但走"纯 workflow"形态的另一个仓库：
[`womenlialtd/llama-sm75-builds`](https://github.com/womenlialtd/llama-sm75-builds) —— 编**官方上游**最新版的 sm_75 包，不含 K2-Horizon。想跟进上游版本用那个，本仓库只在需要 K2-Horizon 时用。

---

*本仓库仅用于个人自用构建。上游 llama.cpp 的版权与维护归 ggml-org / ifm-ai。*
