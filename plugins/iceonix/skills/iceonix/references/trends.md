# 热点与素材

从 YouTube、B站、TikTok、抖音这些公开平台搜热门视频、按播放量排序、分析对标博主，并下载 1080P 视频和双语字幕，供二次创作。全程后台运行，不打开浏览器，**也不调用或唤醒微信**；冰爪不提供微信搜一搜抓取或拓客线索采集。

目前只有 CLI，没有对应的 MCP 工具。

## 命令

```bash
# 关键词搜索最火的 3 个视频，下载视频和双语字幕
iceonix scrape-creator --query "AI Agent" --limit 3 --download

# 指定平台
iceonix scrape-creator --query "AI编程" --platform bilibili --limit 3 --download

# 对标博主：主页链接
iceonix scrape-creator --url "https://www.youtube.com/@Fireship/videos" --limit 5 --download
iceonix scrape-creator --url "https://space.bilibili.com/346157159" --limit 5 --download

# 只看热度和播放量，不下载，几秒生成报告
iceonix scrape-creator --query "Claude" --limit 10

# 只下载音频（1～2MB），用于本机快速转写
iceonix scrape-creator --query "AI热点" --limit 3 --audio-only --download
```

- 自动按播放量从高到低挑选。
- 已下载过的视频会按 `~/Documents/Iceonix/Workspace/Downloads/.download_archive.txt` 跳过，不重复下载。

## 产物

- 视频与字幕：`~/Documents/Iceonix/Workspace/Downloads/Videos/<作者>_[<视频ID>]_<标题>.mp4`，字幕为同名的 `.zh-Hans.vtt` 和 `.en.vtt`。
- 报告：`~/Documents/Iceonix/Workspace/Trending/<日期>_<平台>_<关键词或博主>_trending.md`，frontmatter 里的 `top_videos` 列出每个视频的标题、作者、播放量、时长、链接、本地视频和字幕路径。

## 接下来

- 已有原版字幕：直接读字幕整理逐字稿，拆解爆款的钩子和结构。
- 原创改写：按你自己的人设二次创作，不照搬原文。
- 做成视频：按[做视频](video.md)一章出片。
- 下载的视频是他人作品，只作学习和参考；直接搬运或二次发布前提醒用户注意版权。
