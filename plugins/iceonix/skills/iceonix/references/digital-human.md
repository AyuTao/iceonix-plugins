# 数字人与 AI 视频

冰爪通过用户配置的视频生成网关（用户自己的 GPU 机器或内测网关）生成 AI 视频镜头、做对口型。你负责写分镜、组织素材、提交任务和核对结果；冰爪负责配音、提交、排队、下载和发布。

没有配置网关时，工具会返回「还没有配置视频生成网关」。请用户在冰爪「设置 → AI 与模型 → 视频生成网关」填写地址和 Key，不要尝试别的生成途径。

## 用哪个工具

| 要做的事 | 工具 |
|---|---|
| 固定声音的数字人口播：照片或实拍正脸 + 克隆配音 | `generate_video` 做底片，`lip_sync_video` 对口型 |
| 产品特写、情景过场、空镜 | `generate_video` |
| 角色用模型自己的声音说台词（短剧、情景对话） | `generate_video`，台词写进提示词 |
| 查询进度、取消 | `get_video_generation`、`cancel_video_generation` |

真人录屏和实拍剪辑走[剪辑与达芬奇](editing.md)，用模板做卡点宣传片走[做视频](video.md)。

## 数字人口播

1. **备齐素材。** 口播稿、一到几张场景人像照片（正脸清晰，嘴部不被手、话筒或产品挡住，单人出镜）、产品图、要用的克隆音色（`list_clone_voices` 取 `voiceId`）。照片和音频必须在冰爪工作目录或项目里；用户给的是下载目录等外部文件时，请用户先把文件拖进冰爪项目。只用用户本人或已获授权的人像和声音。
2. **写分镜并确认一次。** 按句把口播稿切段，每段不超过 20 秒（通常一到三句）。每段标明：数字人出镜（用哪张照片、什么情绪）还是产品镜头（用哪张产品图、什么画面）。产品镜头之间穿插数字人出镜，比全程出镜更像真人带货。
3. **配音。** 每段用 `synthesize_speech` 以同一 `voiceId` 合成，不要每段换 `instruct`，否则音色会漂。配完用 `transcribe_video` 回听，读错或句尾被截断就改写重配。
4. **做底片。** 每张场景照片做一次 `generate_video(mode=firstFrame, imagePaths=[照片], aspect=按发布平台, durationSeconds=这张照片下最长一段配音的秒数（上限 20）, seed=固定值)`。提示词写人物对着镜头说话、自然的头部动作和眨眼、镜头固定、不出现文字，例如 `A woman in a navy blazer talks to the camera with a warm smile, natural small head movements and blinking, static medium shot, no text`。同一张照片的多段配音可以共用一个底片，对口型时会自动循环补齐。
5. **对口型。** 每段 `lip_sync_video(videoPath=底片, audioPath=该段配音, quality=high)`。想先给用户看效果，可以挑一段用 `standard` 出样片。
6. **产品镜头。** `generate_video(mode=firstFrame, imagePaths=[产品图], prompt=镜头运动与光线)`。产品镜头自带的声音不要用，剪辑时用对应段的配音盖住。
7. **等结果。** 提交后会立刻返回 `taskId`。底片和产品镜头每 5 秒约需 1 分钟；对口型 high 档约为配音时长的 20 倍，standard 约 10 倍。用 `get_video_generation` 隔一会查一次，绝不换 `requestId` 重投。返回里有 `note`（网关正忙）时任务会自动排队重试，继续等即可。
8. **核对。** 对口型成片用 `transcribe_video` 抽查。嘴型明显对不上、人脸变样或画面闪烁的段落，先换一张更正的照片重做底片，再重新对口型。
9. **拼接。** 由你来剪：按下面「时间线与出片」做成 FCPXML 时间线交给 Final Cut Pro 或达芬奇，或者直接出片。
10. **收进项目并发布。** 用 `add_project_asset` 收进项目，`create_project_work` 建视频作品；发布按[发布与登录](publish.md)。发布时必须打开各平台的 AI 生成声明：`douyinDeclaration`、`kuaishouDeclaration`、`xhsDeclaration`、`wechatDeclaration`、`bilibiliDeclaration` 都传 `ai_generated`。

## 时间线与出片

剪辑由你完成，冰爪不提供拼接工具。按用户要求二选一：

**交给 Final Cut Pro 或达芬奇（用户要精修、加 B-roll 时）。** 写一份 FCPXML 1.11 时间线，存到冰爪工作目录（例如 `AIVideos/<名称>.fcpxml`）。两款软件都能导入：

- Final Cut Pro：有终端时运行 `open -a "Final Cut Pro" <文件>.fcpxml`，否则请用户双击该文件，或在 FCP 里选「文件 → 导入 → XML」。
- 达芬奇：有达芬奇 MCP 时用 `MediaPool.ImportTimelineFromFile(路径, {"importSourceClips": True})` 导入；否则请用户在达芬奇里选「文件 → 导入 → 时间线」。达芬奇免费版对外部脚本可能有限制，导入失败就改用手动导入。

模板（25fps 竖屏；对口型成片是 25fps，`generate_video` 的片段是 24fps，各自声明 format，序列统一用 25fps）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE fcpxml>
<fcpxml version="1.11">
  <resources>
    <format id="seq" frameDuration="1/25s" width="576" height="1024" colorSpace="1-1-1 (Rec. 709)"/>
    <format id="f25" frameDuration="1/25s" width="576" height="1024" colorSpace="1-1-1 (Rec. 709)"/>
    <format id="f24" frameDuration="1/24s" width="576" height="1024" colorSpace="1-1-1 (Rec. 709)"/>
    <asset id="talk1" name="段1 数字人" start="0s" duration="226/25s" hasVideo="1" hasAudio="1" format="f25" audioSources="1" audioChannels="1" audioRate="48k">
      <media-rep kind="original-media" src="file:///绝对路径/段1.mp4"/>
    </asset>
    <asset id="product" name="产品镜头（仅画面）" start="0s" duration="124/24s" hasVideo="1" format="f24">
      <media-rep kind="original-media" src="file:///绝对路径/产品.mp4"/>
    </asset>
    <asset id="voice2" name="段2 配音" start="0s" duration="4s" hasAudio="1" audioSources="1" audioChannels="1" audioRate="48k">
      <media-rep kind="original-media" src="file:///绝对路径/段2.wav"/>
    </asset>
  </resources>
  <library>
    <event name="冰爪数字人">
      <project name="新品数字人口播">
        <sequence format="seq" duration="326/25s" tcStart="0s" tcFormat="NDF" audioLayout="stereo" audioRate="48k">
          <spine>
            <asset-clip ref="talk1" name="段1 数字人" offset="0s" start="0s" duration="226/25s" tcFormat="NDF">
              <asset-clip ref="product" name="B-roll" lane="1" offset="75/25s" start="0s" duration="50/25s"/>
            </asset-clip>
            <asset-clip ref="product" name="段2 产品镜头" offset="226/25s" start="0s" duration="100/25s" tcFormat="NDF">
              <asset-clip ref="voice2" name="段2 配音" lane="-1" offset="0s" start="0s" duration="100/25s"/>
            </asset-clip>
          </spine>
        </sequence>
      </project>
    </event>
  </library>
</fcpxml>
```

要点：

- 主轨（`spine`）按分镜顺序放数字人片段和整段的产品镜头；叠在数字人画面上的 B-roll 用 `lane="1"` 挂在数字人片段下面，`offset` 从该片段开头算起；产品镜头的配音用 `lane="-1"` 挂在产品镜头下面。
- `generate_video` 的片段自带模型生成的声音。做 B-roll 或产品镜头时，给它另外声明一个只写 `hasVideo="1"`、不写音频属性的 asset（如模板里的 `product`），否则达芬奇会把它的声音一起导进来，而 `srcEnable="video"` 达芬奇不认。
- 时间一律写成序列帧数的分数（`帧数/25s`）。片段时长用 `get_video_generation` 返回的成片按 25fps 换算，向下取整，不要超过素材长度。
- `src` 用 `file://` 加绝对路径；路径里的空格、中文按 URL 编码。
- 写完先自查：每个 `ref` 都有对应 asset，主轨各段的 `offset` 首尾相接，`sequence` 的 `duration` 等于主轨总长。

**直接出片（用户不需要精修时）。** 有终端的话用 ffmpeg 按同一顺序拼接：先把各段统一成相同的分辨率、帧率和音频格式，产品镜头用 `-map` 换上对应段的配音，再 concat。成片存到冰爪工作目录的 `AIVideos/`，然后按第 10 步收进项目。这里用 ffmpeg 只是处理你自己生成的视频文件，不违反「只用冰爪 MCP」的规则；但不要改冰爪管理的工程文件。

没有终端、也没有达芬奇 MCP 时，写好 FCPXML 后请用户按上面的方法手动导入。

## 角色说自己的台词

适合短剧和情景对话，声音由模型生成，换不成克隆音色。

1. 用 `generate_video(mode=firstFrame)`，把英文台词写进提示词，例如 `He says: "I want to up the plane!"`。一段约 4 秒效果最好。
2. 生成后用 `transcribe_video` 核对台词，说错就重做。
3. 剪辑时只保留说话的部分，前后各留 0.3 秒。旁白不需要对口型，用 `synthesize_speech` 单独配，盖在没人说话的情景镜头上。

## 限制

- 画幅只有竖屏 576×1024、横屏 1024×576、方形 768×768，单段最长 20 秒。发 1080p 需要后期放大。
- 侧脸、多人同框、嘴部被遮挡时对口型效果差。
- 对口型只改嘴部，表情和头部动作来自底片；想要开心或严肃的表情，在做底片时写进提示词，并配同样情绪的配音。
- 网关一次只跑一个任务，多段会排队；一条 1 分钟的数字人口播通常要半小时左右，先告诉用户预计等待时间。
