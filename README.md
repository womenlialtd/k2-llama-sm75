# k2-llama-sm75

**个人用的 Windows + CUDA sm_75 (GTX 1660) 构建仓库。**

这里是 [`ggml-org/llama.cpp`](https://github.com/ggml-org/llama.cpp) 的一个 fork，**不是官方仓库**，也不是一条开发分支。保留上游代码只有一个原因：GitHub Actions 需要一份源码作为编译对象。

源码本身请去上游看：[ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp)

---

## 我在这个 fork 上做了什么

| 分支 | 内容 |
|---|---|
| `master`（默认分支） | 上游代码 + 我自己加的 1 个 CI workflow 文件，**没有改动 llama.cpp 的任何 C++ 源码** |
| `model/K2Horizon` | 含 K2-Horizon 模型支持的分支，构建固定在这里的 commit `35999d1` |

对上游的改动，全部加起来就一个文件（`ahead_by=4` 是提交数，不是文件数）：

```
.github/workflows/k2-horizon-win-cuda.yml   +393 行
```

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
