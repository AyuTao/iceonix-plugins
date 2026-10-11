# 字幕与配音

转文字、剪后最终字幕、配音、声音克隆和多说话人分离。原素材识别和配音默认在本机运行；用户选择 Gemini 在线转写时会上传音轨，TTS 只有用户已选局域网或云端提供方时才出网。

转写、配音可能要几十秒到几分钟，中间不逐步播报，只在需要确认输入文件时和给最终结果时说话。

## 工具

| 用途 | MCP 工具 | CLI（仅没有 MCP 时） | 产物 |
|---|---|---|---|
| 语音转字幕 | `transcribe_video` | `iceonix transcribe` | 字幕段（start/end/text），可带词级时间 |
| 剪后最终字幕 | `generate_final_subtitles` / `get_final_subtitles` / `cancel_final_subtitles` | — | 剪后正文的字幕和词表 |
| 查看 TTS 提供方 | `list_tts_providers` | — | 配置与密钥是否就绪，不返回密钥 |
| 查看音色 | `list_tts_voices` | `iceonix voices list --provider local\|elevenlabs` | 本地克隆或云端账号音色 |
| 查看保存的克隆音色 | `list_clone_voices` | `iceonix voices list` | 稳定 `voiceId` |
| 文字转语音、声音克隆 | `synthesize_speech` | `iceonix tts` | WAV 音频 |
| 多说话人分离 | — | `iceonix diarize` | 说话人区间 |
| 下载模型 | — | `iceonix download-model <key>` | — |

识别、配音和分离这类确定性命令可以由本机 CLI 直接运行，不要求冰爪 App 常开；读剪辑工程和最终字幕依赖运行中的 App。CLI 装在 `/usr/local/bin/iceonix`，需要 macOS 15 以上和 Apple Silicon。

## 模型

命令依赖的模型要先下载。不确定时先 `iceonix download-model <key>`：已下载会立即返回，未下载会自动下，大模型要几分钟。也可以让用户在冰爪「音频工坊 → 模型仓库」里点下载。

| 用法 | 需要的模型 key |
|---|---|
| 转写（默认 qwen3） | `qwen3_asr` + `qwen3_aligner` |
| 转写 `--engine gemini` | 无本地模型；需要用户在冰爪里配置的 Gemini |
| 转写带说话人 / `diarize` | 另加 `sortformer` |
| `diarize --asr` | `sortformer` + `qwen3_asr` + `qwen3_aligner` |
| 本地配音 | `voxcpm2`（中文、克隆、文字造音色）或 `kokoro`（英文） |
| 局域网 / ElevenLabs / Deepgram 配音 | 无本地模型；需先在冰爪里保存提供方配置 |

报「模型尚未下载」时，告诉用户模型大小并问是否现在下载，同意后下载再重试原命令。不要伪造结果。

## 转字幕

```bash
iceonix transcribe --video "/绝对路径/a.mov" --language zh-CN --engine qwen3 --diarize
```

- 语言：`zh-CN` / `en-US` / `ja-JP` / `ko-KR`。`--diarize` 可选，给每条字幕标说话人 `[S1]` / `[S2]`。
- 输出 JSON 字幕段。用户要 SRT/VTT 文件时，请他在冰爪音频工坊里导出，或你按 JSON 拼好再给他。

**词级时间用于剪辑：** `transcribe_video(videoPath, engine="gemini", includeWords=true)`（CLI 加 `--include-words`）。默认引擎仍是本地 qwen3，只有用户选了在线时才用 gemini。Gemini 固定用逐字原话（VERBATIM）词表，最长 30 分钟；结果的时间都是源素材秒数。识别结果本身不改音视频，删哪些段由你按原话决定，再经剪辑工具应用。整理过的 SMART 文本只能当参考，不能套用旧的逐词时间。

## 剪后最终字幕

用于冰爪录制工程剪完后的正式字幕。用户先完成或授权剪辑，再走这里；不用先导出整片，也不要借机新增删段。

1. `get_editor_project` 读最新 `revision`、当前 `document.subtitles` 和 `finalSubtitlesCurrent`。切点始终以 `source` 原始词级为准；已有最终字幕时，原始词表在 `originalTranscript`。不能把 `editedContent` 的秒数当作删段、追焦或鼠标事件的时间。
2. 生成一个 UUID 作 `requestId`，调用 `generate_final_subtitles(projectPath, requestId, expectedRevision, engine, localeID)`。**这个工具默认 `gemini`**，需要用户已选择在线转写或授权；要本地就显式传 `engine="qwen3"`。`localeID` 默认 `auto`。
3. 冰爪按删段和出入点导出剪后正文的原声，带上麦克风和系统音量，不混片头、背景音乐和点击声，再转写。
4. 之后只用 `get_final_subtitles(projectPath, requestId)` 查同一个任务，取消用 `cancel_final_subtitles`。状态有 `running / completed / cached / stale / failed / cancelled / unknown`。不得因等待、超时或 `unknown` 换新 ID 重投；同一工程同时只能跑一个任务。
5. `completed` 或 `cached` 后重新读工程，再继续编辑或导出。这时字幕的 `timeBasis="editedContent"`，从剪后正文 0 秒开始。默认复用相同剪辑、引擎和语言的有效结果；`force=true` 才重新识别，在线可能再次计费。
6. 删段、出入点、麦克风或系统音量、原素材变化会让最终字幕过期，要重新生成；追焦、光标、字幕样式、背景音乐、背景和封面不会。生成期间工程变了，任务标记 `stale`，结果单独保留，不覆盖工程。
7. 用 `apply_editor_edit` 修改或移除最终字幕时，`subtitle` / `remove_subtitle` 操作必须显式传 `timeBasis="editedContent"` 并用最终字幕的时间。不要直接改字幕 JSON，也不要猜时间。

这条链路仍在测试，汇报时只说生成了什么，不要说成已经完整验收。

## 配音与克隆

```bash
iceonix tts --text "要合成的文字" --output "/绝对路径/out.wav" \
  --provider local --engine voxcpm2 --language chinese --voice-id "稳定 UUID"
```

- **保存的克隆音色**：先 `list_clone_voices`（CLI `iceonix voices list`）按名称选 `id`，再给 `synthesize_speech` 传 `voiceId`。不要猜文件路径。
- **一次性参考音**：`--clone-ref /绝对路径/ref.wav`，建议再加 `--ref-text "参考音的逐字稿"`。参考音 3～5 秒、单人、无音乐和混响最好；超过 5 秒只取前 5 秒并忽略 `--ref-text`。
- **英文**：`kokoro` 只用固定 `--speaker`，默认 `af_heart`。
- **网络 TTS**：先 `list_tts_providers` 确认配置；ElevenLabs 再用 `list_tts_voices(provider=elevenlabs)` 选音色，然后 `synthesize_speech(provider=lan|elevenlabs|deepgram)`。密钥只由用户在冰爪里保存，不要让用户发给你。
- **Gemini 配音**：`synthesize_speech(provider=gemini)` 用用户自己的 Gemini Key；`list_tts_voices(provider=gemini)` 列预置音色（默认 Kore），`remoteVoice` 选音色。
- **长文本**：VoxCPM2 自动按每段最多 80 字切分并对齐响度。
- **整篇保持同一音色（铁律）**：音色只在一次请求内稳定。整篇旁白用一次 `--text-file` 合成；确需分段调用时，每次都传同一个 `voiceId` 或同一份参考音。不要分多次只带 `--instruct`，每次都会变成不同的声音。
- 参数不兼容被拒绝时按错误改参数，不要删掉被拒的参数后假装音色没变。

## 多说话人分离

```bash
iceonix diarize --input "/绝对路径/talk.mov"                         # 只分人
iceonix diarize --input "/绝对路径/talk.mov" --asr --language zh-CN  # 分人并转写
```

## 拿到输入文件

聊天通道（如 Hermes 的微信、飞书）里，用户发的音视频可能只显示成占位符，文件其实已下载到本地缓存。不要卡住让用户给路径，自己找最新文件：

```bash
find ~/.hermes/cache/videos ~/.hermes/cache/documents ~/.hermes/cache/audios -type f \
  \( -iname "*.mov" -o -iname "*.mp4" -o -iname "*.wav" -o -iname "*.mp3" -o -iname "*.m4a" \) 2>/dev/null \
  -exec stat -f '%Sm %z %N' -t '%H:%M:%S' {} \; | sort -r | head -5
```

用最新文件前，先把文件名、时间和大小报给用户确认；连发了多个就列出来让他选。

## 汇报

- 告诉用户产物路径，或直接把音频、字幕内容发给他。产物默认写在用户指定目录或 `/tmp`。
- 本章只产出本地音频和字幕文件，不替用户发布；要发布按[发布与登录](publish.md)做。

模型 key：`qwen3_asr` `qwen3_aligner` `voxcpm2` `kokoro` `sortformer`。`deepfilternet` `sidon` `flashsr` `sourcesep` 只供冰爪内部清理参考音和提取人声。
