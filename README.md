# YouTube 频道全量知识库构建指南 (NotebookLM 版)

本笔记记录了将 YouTube 视频（以 Dan Koe 频道 235 个视频为例）转化为个人 AI 知识库的完整工程实践。

## 1. 技术流程 (Technical Workflow)

### Step 1: 工业级字幕爬取

**方案说明**：使用 `yt-dlp` 绕过不稳定的网页 DOM 解析，直接抓取自动生成的字幕流。加入随机延迟以规避反爬机制。

```python
import yt_dlp

def download_channel_transcripts(playlist_url):
    ydl_opts = {
        'skip_download': True,         # 仅提取文本，不下载视频
        'writeautomaticsubs': True,    # 强制抓取自动生成的字幕
        'subtitleslangs': ['en'],      # 限制仅英文，减少请求量规避 429 错误
        'sleep_requests': 1,           # 关键：API 请求间延迟 1 秒
        'sleep_interval': 3,           # 视频下载间随机延迟 3-10 秒
        'max_sleep_interval': 10,
        'outtmpl': 'notebook_lm_data/%(title)s.%(ext)s',
        'ignoreerrors': True,          # 遇到个别视频错误不中断执行
    }
    with yt_dlp.YoutubeDL(ydl_opts) as ydl:
        ydl.download([playlist_url])

# 填入博主的 Uploads Playlist URL
download_channel_transcripts("https://www.youtube.com/playlist?list=UUWXYDYv5STLk-zoxMP2I1Lw")

```

### Step 2: 数据去重与深度清洗

**方案说明**：利用 PowerShell 管道流处理。针对同一视频产生的多个字幕版本（如 `.en.vtt` 和 `.en-orig.vtt`）进行去重，并剔除元数据。

```powershell
# 在数据存放目录下运行
Get-ChildItem *.vtt | Group-Object { $_.BaseName.Split('.')[0] } | ForEach-Object {
    $_.Group[0] | Get-Content | Select-String -Pattern "^(?!\d{2}:|\d{1}:|WEBVTT|Kind:|Language:).+"
} | Out-File -FilePath Full_Library.txt -Encoding utf8

```

### Step 3: 大文件物理切分

**方案说明**：NotebookLM 对单文件解析有潜在的缓冲区限制，将大文件切分为 10MB 左右的切片可确保 100% 上传成功。

```python
def split_text_file(filename, lines_per_file=50000):
    with open(filename, 'r', encoding='utf-8') as f:
        content = f.readlines()
    for i in range(0, len(content), lines_per_file):
        with open(f'Knowledge_Part_{i//lines_per_file}.txt', 'w', encoding='utf-8') as out:
            out.writelines(content[i:i+lines_per_file])

split_text_file('Full_Library.txt')

```

### Step 4: NotebookLM 挂载

* 将生成的 `Knowledge_Part_x.txt` 批量上传至 [NotebookLM](https://notebooklm.google.com/)。
* **反馈**：AI 对保留了时间轴的原始文本具备极强的鲁棒性，无需过度清洗。

---

## 2. 对话式 Debug 记录 (Q&A)

> **Q: 为什么最初的爬虫脚本会报 `AttributeError`？**
> **A**: 传统的网页爬虫依赖对 HTML 结构的硬解析。YouTube 经常更新页面 DOM 或采用动态加载，导致解析器找不到属性。改用 `yt-dlp` 这种基于协议的工具能直接获取字幕流，更稳健。

> **Q: 为什么必须在命令中增加 `sleep`（延迟）？**
> **A**: 为了规避 **HTTP Error 429 (Too Many Requests)**。高频请求会触发服务器的防护机制，增加随机延迟能模拟真人操作，保护 IP。

> **Q: 为什么合并后的 TXT 里会有重复内容？**
> **A**: YouTube 往往为同一个视频提供多个语言标记（en, en-orig 等）。合并脚本通过 `Group-Object` 逻辑，确保每个视频 ID 只被提取一次。

> **Q: 34MB 的纯文本为什么上传失败？**
> **A**: Web 端在解析超大文本时容易出现超时或内存溢出。物理切分是解决大规模数据录入 RAG 系统最简单有效的工程手段。

---

## 3. 经验总结 (Lessons Learned)

1. **工程直觉优先**：当轻量级库（如 `youtube-transcript-api`）报错时，果断转向工业级工具（`yt-dlp`）。
2. **数据不必完美**：AI 具备极强的“抗噪”能力，时间戳等元数据不影响语义理解，甚至能作为时空参考。
3. **技术隔离伤害**：利用 AI 辅助写作和整理，可以有效隔离外界负面反馈的心理伤害，释放输出压力。

