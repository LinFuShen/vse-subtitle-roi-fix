# VSE Subtitle ROI Fix（非官方 Fork）

项目介绍、安装和使用说明请看[原项目 YaoFANGUK/video-subtitle-extractor](https://github.com/YaoFANGUK/video-subtitle-extractor#readme)。

由于上游相关 Issue 暂未有人接手，这里 Fork 了原项目，在默认 `main` 分支提供修复。这不是官方发行版。

[下载修复源码 ZIP](https://github.com/LinFuShen/vse-subtitle-roi-fix/archive/refs/heads/main.zip)（源码，不是安装包）。

## 修复现象：字幕明明在框内，却被全部丢弃

Windows 快速模式下，只框住画面底部的字幕、不框上方水印，原版仍可能把正常字幕全部判成「在选区之外」，导出空 SRT。本机用一段测试视频复现：开启原有选区过滤时，连高置信度的框内字幕也被误判，最终没有保留任何字幕。**只框字幕是正确操作，不需要把框拉大到包住水印。**

旧流程中，VSF 用自定义框找时间点，但 OCR 会重新读原视频帧，并可能按默认「下半屏」设置先裁一次。这样 OCR 返回的坐标是相对于裁剪图的，过滤时却拿它与原图上的自定义框比较，导致框内字幕被误判。直接跳过过滤虽能导出文本，却会让框外水印进入结果。

修复后，有自定义框时直接按框裁原始帧再 OCR，并把识别框坐标还原到原图，保留原有的位置和置信度检查。用同一段测试视频、开启过滤复测，成功导出非空 SRT：框内字幕得到保留，未检出框外水印文字。

## 修改文件清单

相对上游基线 [`85746f7`](https://github.com/YaoFANGUK/video-subtitle-extractor/commit/85746f7)，本仓库的 `main` 分支共修改 **4 个文件**：

| 文件 | 改动 | 目的 |
| --- | --- | --- |
| [`backend/tools/subtitle_ocr.py`](backend/tools/subtitle_ocr.py) | 有自定义字幕框时，跳过「全屏／上半／下半」的默认裁剪，改为直接从原始帧按自定义框裁图。OCR 后将文字框坐标换算回原图，再走原有的选区和置信度检查。无自定义框时维持原行为。 | 框外水印不进入这一路 OCR，也不会把裁剪后的相对坐标直接拿去和原图选区比较。 |
| [`backend/tools/ocr.py`](backend/tools/ocr.py) | 将文字框来源从 `dt_polys` 换成与 `rec_texts`、`rec_scores` 对应的 `rec_polys`。 | 避免检测框和识别文本错配，影响选区过滤。 |
| [`backend/main.py`](backend/main.py) | 将 `SubtitleExtractor.manage_process(pid)` 改成 `manage_process(self, pid)`。 | 修复命令行调用实例方法时因缺少 `self` 触发的 `TypeError`。 |
| [`README.md`](README.md) | 链接原项目，记录 Fork 原因、文件改动和 Windows CPU／CUDA 排障经验。 | 方便核对本仓库的 `main` 分支与上游的差异。 |

本仓库的 `main` 分支未修改 GUI、默认配置、依赖清单或 VSF 参数。本机试过的 CPU／oneDNN 规避方案和调试参数没有提交到本仓库的 `main` 分支。

## 目前发现的其他问题及相应解决方案

以下问题和解决方案来自 Windows 环境下的测试；不代表这些问题都已在本仓库中修复。

- **CPU / oneDNN：** 本机用 PaddlePaddle 3.3.1 CPU 版开启 oneDNN（`enable_mkldnn=True`）时，OCR 推理触发了 [Paddle #77340](https://github.com/PaddlePaddle/Paddle/issues/77340) 的 `ConvertPirAttribute2RuntimeAttribute` 错误。用 PP-OCRv5 mobile 单帧验证过两种规避方式：关闭 oneDNN，或在独立环境用 `paddlepaddle==3.2.2` 保留 oneDNN 加速。两者都不是本仓库的 `main` 分支的默认配置；CPU 整集提取未做回归。
- **NVIDIA GPU：** 有 NVIDIA 显卡、勾选「硬件加速」，不代表 CPU 版 Paddle 会使用 GPU。按[上游安装说明](https://github.com/YaoFANGUK/video-subtitle-extractor#readme)和 [Paddle 官方安装指南](https://www.paddlepaddle.org.cn/install/quick)选与驱动、Python 和系统兼容的 `paddlepaddle-gpu`，按所选包的要求安装或补齐 CUDA／cuDNN 运行库。建议新建环境，不在同一环境混装 CPU 和 GPU 版 Paddle。`nvidia-smi` 显示的 CUDA 版本是驱动支持的上限，不代表对应运行库已安装；也不必在 wheel 已提供所需运行库时重复安装完整 Toolkit。
- **确认 GPU 真的在用：** 在 VSE 的 Python 环境确认 `paddle.device.is_compiled_with_cuda()` 为 `True`、`paddle.static.cuda_places()` 能看到设备，再开启「硬件加速」跑一小段字幕。Windows 上用 PaddlePaddle GPU 3.3.1（CUDA 12.6）和快速模式完成了整集验证；不代表其他驱动或依赖组合也已验证。
- **进度卡住或输出为空：** 先查实际日志。CPU 推理报错、VSF 临时图写入失败、选区过滤丢弃文本是不同问题，不能只凭百分比判断。调试时不要手动删除正在使用的输出目录，也不要为了出结果直接跳过选区过滤，否则框外台标会混入字幕。
