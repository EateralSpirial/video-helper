# 处理管线与架构

## 1. 架构目标

`video-helper` 的核心不是某个单一 AI 模型，而是一条可复现、可审计、可替换模块的媒体处理管线。

P0 的关键原则：

> AI 或检测模型负责提出区间和文本判断；确定性的时间轴引擎负责执行剪切、重映射和渲染。

## 2. P0 总体管线

```text
原始视频
   │
   ▼
[1] Ingest / 媒体检查
   │
   ├── media.json
   └── proxy.mp4
   │
   ▼
[2] Audio / 音频提取与标准化
   │
   ├── analysis.wav
   └── audio_report.json
   │
   ├───────────────────────┐
   ▼                       ▼
[3A] VAD / 静音分析      [3B] ASR / 转录
   │                       │
   ├── speech_segments     ├── transcript
   └── silence_segments    └── word timestamps
   │                       │
   └───────────┬───────────┘
               ▼
[4] Planner / 剪切计划
               │
               ├── cut_plan.json
               └── cut_report.md
               │
               ▼
[5] Timeline / 时间轴映射
               │
               └── timeline_map.json
               │
       ┌───────┴────────┐
       ▼                ▼
[6A] Subtitle       [6B] Renderer
     remap               │
       │                 │
       ├── SRT/ASS       ├── rough_cut.mp4
       └── JSON          └── subtitled.mp4
       │                 │
       └────────┬────────┘
                ▼
[7] QC / 质量检查
                │
                ├── qc_report.json
                └── qc_report.md
```

## 3. 为什么先分析原始时间轴

P0 应在原始音频上完成 VAD 和 ASR，再生成成片时间轴。

这样可以：

- 保留完整原始证据；
- 清楚解释每个删除区间；
- 让字幕、章节和后续语义分析共享同一原始坐标；
- 允许用户恢复任何被删除内容；
- 避免多次剪切后时间戳来源混乱。

渲染完成后，可选地对成片进行二次 ASR 校验，但二次结果不能替代原始时间轴数据。

## 4. 模块边界

### 4.1 Ingest

职责：

- 验证输入；
- 获取媒体元数据；
- 初始化项目；
- 生成代理文件；
- 管理内容哈希。

不得负责：

- 判断静音；
- 生成字幕；
- 决定剪辑。

### 4.2 Audio Analyzer

职责：

- 提取分析音轨；
- 生成响度、削波和波形统计；
- 提供统一音频格式。

### 4.3 Speech Detector

职责：

- 输出语音、非语音和低置信度区间；
- 合并碎片；
- 应用前后缓冲；
- 保留原始检测结果和规则处理后的结果。

可替换实现包括传统响度门限、VAD 模型或两者融合。

### 4.4 Transcriber

职责：

- 生成句级和词级转录；
- 保存置信度；
- 统一文本和时间戳格式；
- 支持术语词典与语言配置。

### 4.5 Cut Planner

职责：

- 读取语音区间、静音区间和配置；
- 生成 `keep`、`compress`、`drop` 决策；
- 处理缓冲、最短片段和相邻切口；
- 输出预计时长变化；
- 保证计划无重叠、连续且确定。

不得直接调用 FFmpeg 修改视频。

### 4.6 Timeline Mapper

职责：

- 把原始时间轴映射到成片时间轴；
- 提供双向查询；
- 映射字幕、章节、标记和后续视觉事件；
- 统一处理浮点精度与累计误差。

这是 P0 最重要的基础模块之一。

### 4.7 Subtitle Builder

职责：

- 将转录映射到成片；
- 断句；
- 合并过短字幕；
- 拆分过长字幕；
- 生成 SRT、ASS 和 JSON；
- 检查时间戳有效性。

### 4.8 Renderer

职责：

- 根据剪切计划生成视频和音频；
- 在切口处应用短音频交叉淡化；
- 烧录可选字幕；
- 输出净版；
- 管理编码器选择和回退。

P0 首选基于 FFmpeg 构建确定性执行层。

### 4.9 Quality Checker

职责：

- 校验输出可解码；
- 校验音画与字幕时长；
- 检查黑帧、异常静音、字幕越界和时间漂移；
- 输出需人工审阅的具体时间点。

## 5. 项目目录

```text
<project>/
├── project.json
├── config.yaml
├── source/
│   └── original.<ext>
├── cache/
│   ├── proxy.mp4
│   ├── analysis.wav
│   └── hashes.json
├── analysis/
│   ├── media.json
│   ├── audio_report.json
│   ├── speech_segments.raw.json
│   ├── speech_segments.json
│   ├── silence_segments.json
│   ├── transcript.json
│   ├── transcript.txt
│   └── transcript.md
├── edit/
│   ├── cut_plan.json
│   ├── cut_report.md
│   └── timeline_map.json
├── subtitles/
│   ├── subtitles.json
│   ├── subtitles.srt
│   └── subtitles.ass
├── preview/
│   └── preview.mp4
├── export/
│   ├── rough_cut.mp4
│   ├── rough_cut_subtitled.mp4
│   └── clean_audio.wav
└── reports/
    ├── qc_report.json
    └── qc_report.md
```

原始媒体可以采用以下任一策略：

- 复制到 `source/`；
- 建立只读链接；
- 仅在 `project.json` 中记录绝对路径。

默认策略应优先避免无必要复制超大文件，同时明确提示移动原文件会使项目失效。

## 6. 核心数据契约

所有 JSON 文件必须包含：

```json
{
  "schema_version": "0.1.0",
  "created_at": "ISO-8601 timestamp",
  "producer": {
    "name": "module-name",
    "version": "tool-version"
  },
  "source_hash": "sha256-or-equivalent"
}
```

### 6.1 时间表示

内部统一使用整数时间单位，建议采用微秒或纳秒。

对外 JSON 可以同时提供：

```json
{
  "start_us": 83420000,
  "end_us": 91160000,
  "start_seconds": 83.42,
  "end_seconds": 91.16
}
```

内部决策禁止仅依赖二进制浮点秒数，以降低长视频累计漂移。

### 6.2 speech_segments.json

```json
{
  "segments": [
    {
      "id": "speech-000001",
      "start_us": 1200000,
      "end_us": 14800000,
      "kind": "speech",
      "confidence": 0.98,
      "detector": "vad",
      "padding_before_us": 180000,
      "padding_after_us": 220000
    }
  ]
}
```

### 6.3 transcript.json

```json
{
  "language": "zh",
  "segments": [
    {
      "id": "asr-000001",
      "start_us": 1310000,
      "end_us": 4820000,
      "text": "一个系统，就是多个相互连接的部分。",
      "confidence": 0.96,
      "words": [
        {
          "text": "一个",
          "start_us": 1310000,
          "end_us": 1640000,
          "confidence": 0.97
        }
      ]
    }
  ]
}
```

### 6.4 cut_plan.json

```json
{
  "preset": "conservative",
  "source_duration_us": 3600000000,
  "estimated_output_duration_us": 2840000000,
  "decisions": [
    {
      "id": "decision-000001",
      "source_start_us": 0,
      "source_end_us": 1200000,
      "action": "drop",
      "reason": "leading_non_speech",
      "confidence": 0.99,
      "locked": false
    },
    {
      "id": "decision-000002",
      "source_start_us": 1200000,
      "source_end_us": 14800000,
      "action": "keep",
      "reason": "speech",
      "confidence": 0.98,
      "locked": false
    }
  ]
}
```

计划校验规则：

- 所有区间按起点升序；
- 区间互不重叠；
- 全部决策共同覆盖完整原始时间轴；
- `keep` 保持原时长；
- `compress` 必须具有目标时长；
- `drop` 的目标时长为零；
- 锁定决策不会被自动重算覆盖。

### 6.5 timeline_map.json

```json
{
  "segments": [
    {
      "source_start_us": 1200000,
      "source_end_us": 14800000,
      "output_start_us": 0,
      "output_end_us": 13600000,
      "mapping": "linear"
    }
  ]
}
```

## 7. 配置模型

建议配置分层：

1. 内置默认值；
2. 预设；
3. 用户级配置；
4. 项目配置；
5. 命令行覆盖。

优先级从低到高。

示例：

```yaml
preset: conservative

speech_detection:
  min_speech_ms: 180
  min_non_speech_ms: 700
  merge_gap_ms: 250
  padding_before_ms: 180
  padding_after_ms: 220

cutting:
  short_gap_action: keep
  long_gap_action: compress
  compressed_gap_ms: 500
  minimum_clip_ms: 900

transcription:
  language: zh
  word_timestamps: true
  glossary: []

subtitles:
  max_chars_per_line: 18
  max_lines: 2
  min_duration_ms: 700
  max_duration_ms: 6500

render:
  preview_height: 540
  burn_subtitles: true
  preserve_source_fps: true
```

这些数值只是初始工程默认值，应通过真实录播样本测试调整，并允许预设覆盖。

## 8. 预设

### conservative

- 保留更多停顿；
- 对低置信度区间默认保留；
- 主要压缩明显长空白；
- 适合首轮使用和正式课程。

### standard

- 适度压缩句间空白；
- 更积极合并片段；
- 适合普通知识口播。

### aggressive

- 更短的目标停顿；
- 允许处理更多边缘无语音区间；
- 必须先生成预览；
- 适合节奏较快的短内容。

## 9. CLI 设计

```bash
video-helper init input.mp4 --project ./projects/episode-001
video-helper analyze ./projects/episode-001
video-helper plan ./projects/episode-001 --preset conservative
video-helper inspect ./projects/episode-001
video-helper preview ./projects/episode-001
video-helper render ./projects/episode-001 --with-subtitles
video-helper run input.mp4 --preset conservative
```

推荐全局参数：

```text
--config <path>
--force
--resume
--no-cache
--log-level
--json
--device cpu|cuda|auto
```

## 10. 状态与缓存

每个步骤应具备：

- 输入依赖哈希；
- 配置哈希；
- 工具版本；
- 开始、结束和失败状态；
- 输出文件清单；
- 错误摘要。

当输入、配置和版本均未变化时，允许复用结果。

任何步骤失败后，之前的合法输出应保留。

## 11. 跨平台策略

### Windows

- 首要支持环境；
- 处理包含中文、空格和长路径的文件名；
- 避免依赖 Bash；
- 子进程参数使用参数数组，避免拼接 Shell 字符串；
- 提供 FFmpeg 可执行文件发现与诊断。

### Ubuntu

- 支持 GPU 推理和批处理；
- 支持无图形界面运行；
- 适合作为性能测试与长任务环境。

### macOS

- 支持 Apple Silicon；
- 基础 P0 管线必须可运行；
- 硬件加速作为可选优化。

## 12. 测试策略

### 单元测试

- 区间合并；
- 缓冲裁剪；
- 剪切计划完整覆盖；
- 时间轴正向与反向映射；
- 字幕拆分与合并；
- 配置优先级；
- 缓存失效规则。

### 集成测试

准备小型合成视频：

1. 开头 2 秒静音；
2. 5 秒语音；
3. 0.3 秒停顿；
4. 5 秒语音；
5. 4 秒长静音；
6. 5 秒语音；
7. 结尾 2 秒静音。

验证：

- 短停顿保留；
- 长静音压缩；
- 首尾空白删除；
- 字幕重映射正确；
- 输出总时长符合计划。

### 回归测试

建立脱敏样本集，覆盖：

- 安静室内录音；
- 空调背景噪声；
- 键盘和鼠标声音；
- 低音量说话；
- 快速连续口播；
- 中英混合；
- 一小时以上长视频。

## 13. 后续架构扩展

P1—P3 模块应作为独立消费者接入 P0 数据：

```text
transcript.json
cut_plan.json
 timeline_map.json
subtitles.json
```

例如：

- P1 语义编辑器修改 `cut_plan.json`；
- P1 摘要器读取映射后的转录；
- P2 知识图谱生成器读取章节和字幕；
- P2 动画引擎向统一时间轴增加视觉轨道；
- P3 多模态导演生成候选计划，但仍通过同一 Planner、Timeline 和 Renderer 执行。

这样可以确保后续智能能力持续增强，同时不破坏 P0 的确定性基础。