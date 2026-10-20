# 截图所示近期更新项目：中文导读

本页补充用户截图中显示最近约两天内更新的四个项目：DeepInfer、DeepGEMM、deepseek-recipe、DeepJIT。截图里的 Updated 是仓库更新时间，不能据此认定文章或项目首次发布日期。以下根据实际读取的官方 README 整理，不是逐字翻译，也不是对最近某个提交的变更总结。

## 项目导航

| 项目 | 官方资料 | 核心作用 |
| --- | --- | --- |
| DeepInfer | [README](https://github.com/deepseek-ai/DeepInfer#readme) | 大规模模型推理框架 |
| DeepGEMM | [README](https://github.com/deepseek-ai/DeepGEMM#readme) | GPU 矩阵运算及融合计算内核 |
| deepseek-recipe | [README](https://github.com/deepseek-ai/deepseek-recipe#readme) · [中文](https://github.com/deepseek-ai/deepseek-recipe/blob/main/README.zh.md) | API 协议转换、提示编码和输出解析 |
| DeepJIT | [README](https://github.com/deepseek-ai/DeepJIT#readme) | 设备内核即时编译、缓存和加载 |

## DeepInfer：把模型推理组织成高效执行流程

官方描述为支持 DeepSeek 大规模生产服务的高性能推理框架，本次公开版本提供离线推理及规模化服务所需的核心能力。当前 README 列出的模型是 DeepSeek-V4.1-Flash，硬件为 NVIDIA SM100 GPU 或华为 Ascend 950 NPU；基础环境要求 Linux、GCC 13+、Python 3.12，以及对应的 CUDA 13 或 CANN 9.2。

重点包括注意力与专家并行、专家负载均衡、异步批调度、草稿 token 与置信度调度验证、结构化输出等。它集成 DeepGEMM、DeepEP、FlashMLA 等底层组件。公开版本仍不包含完整服务所需的 KV cache 系统、完整的分离部署方案与跨副本负载均衡，不能直接等同于开箱即用的完整线上服务。

**阅读任务：** 区分 prefill 与 decode；说明批处理吞吐量、单用户速度和延迟为何要同时衡量。先阅读 examples/offline_inference.py 及 examples/README.md，不急于下载权重。

## DeepGEMM：加速矩阵运算与 MoE 计算

当前项目已不只是单一 GEMM 函数库：README 涵盖 FP8、FP4、BF16 矩阵计算、融合通信的 Mega MoE、索引器相关计算与其他算子。设备内核运行时通过 DeepJIT 编译；这不代表安装完全不需要构建主机侧 C++ 扩展。

GEMM 是神经网络线性层的核心计算。针对 MoE，不同专家接收的 token 数量不同，分组矩阵计算和数据布局会直接影响效率。Mega MoE 进一步融合专家分发、线性层、激活与结果合并，使通信与计算重叠。

**实践任务：** 用 NumPy 比较逐行循环与一次矩阵乘法的耗时，并验证结果接近。这只是帮助理解矩阵计算组织方式，不是 DeepGEMM 性能测试。真实内核实验需满足官方 NVIDIA SM90/SM100、CUDA 12.9+ 等要求。

## deepseek-recipe：把不同 API 转换为模型可理解的对话

项目由 Rust 库和 Python 绑定组成，把不同格式的 API 请求转换成统一对话结构，再编码为 DeepSeek 模型提示；也能把模型输出解析为相应协议的响应。其职责是协议和数据表示转换，不是模型推理引擎。

官方示例涉及 Chat Completions、Responses、Messages、工具定义、结构化输出，以及图像预处理。结构化约束会被传递给调用方，实际是否通过约束解码执行由后端决定，协议转换本身不保证模型输出满足约束。

**实践任务：** 先阅读 README 中 Python 的 ChatCompletionRequest → convert → render_conversation 示例，跟踪角色、消息内容和工具定义如何进入最终提示。官方 server-py 和 server-rs 使用模拟推理，适合先理解协议流程。模型和图像支持依具体版本而异。

## DeepJIT：运行时编译与缓存设备内核

DeepJIT 是面向 NVIDIA CUDA GPU 与华为 Ascend NPU 的轻量级、仅头文件 C++20 JIT 运行时，提供编译、缓存、加载和启动设备内核的统一流程。缓存键考虑源码、跟踪的头文件、编译器版本及选项等信息，以便复用编译结果。

它解决的是内核库的基础设施问题，不是自动把任意 Python 代码变成 GPU 程序。当前 README 将“直接从 Python 传入内核源码的编译 API”和“根据历史进行缓存预热”标为开发中，不能按已可用功能使用。

**实践任务：** 设计一个简化编译缓存：输入为源码、编译选项和编译器版本，输出为哈希键；解释漏掉其中任意一项为什么可能错误复用产物。该练习可在 CPU 上完成，真实编译仍需相应设备工具链。

## 面向有编程基础的学习安排

1. 第一次：请求到输出的完整路径。以 recipe 的请求转换为入口，画出协议适配、推理调度、设备内核、输出解析的关系；练习识别各层输入输出。
2. 第二次：矩阵乘法、MoE 与 DeepGEMM。完成 CPU 矩阵实验，再读 grouped GEMM 与 Mega MoE。
3. 第三次：DeepJIT 的编译和缓存。完成缓存键练习，对照官方实现理解失效条件。
4. 第四次：DeepInfer 的 prefill/decode、批处理和并行。结合前面组件解释端到端延迟。
5. 后续回到 [V3 与 R1 报告](README.md)，把系统工程与模型架构、强化学习联系起来。

以上为学习计划，未声称已经运行上游示例或完成 GPU/NPU 性能验证。实际学习时按可用硬件选择实验。

## 来源范围

摘要依据四个官方仓库 main 分支 README；上游内容会继续变化。截图还列出了 deepseek-harness（3 天前更新）和 DeepEP（4 天前更新），本次未纳入“两天内”的四项范围。需要扩展时可单独整理。
