# arXiv Search（arXiv 论文助手）

KiraAI 插件：arXiv 论文查询、翻译与下载。数据源为 arXiv 官方 API
`export.arxiv.org/api/query`（Atom XML 格式），PDF 下载自 `arxiv.org/pdf/{id}`，
LaTeX 源码下载自 `arxiv.org/e-print/{id}`。

## 功能

| 命令 | 说明 |
| --- | --- |
| `/arxiv search <关键词> [-t]` | 搜索论文，返回标题、摘要（截断）、作者、arXiv ID、分类、PDF 链接，默认 5 条；加 `-t` 附带标题译文 |
| `/arxiv get <arXiv ID> [-t]` | 获取单篇论文完整详情（标题/全部作者/摘要/日期/分类/PDF）；加 `-t` 附带标题与摘要译文 |
| `/arxiv tr <arXiv ID>` | 将单篇论文的标题与摘要翻译成中文（使用默认 LLM，失败友好兜底） |
| `/arxiv dl <arXiv ID> [多个ID]` | 下载 PDF 到 `data/files/arxiv_pdf/` 并返回本地路径 |
| `/arxiv src <arXiv ID>` | 下载 LaTeX 源码包（e-print，tar.gz/tex.gz/tex）到 `data/files/arxiv_src/` 并返回本地路径 |
| `/arxiv help` | 查看帮助 |

同时注册 6 个 LLM 工具：`arxiv_search`、`arxiv_get`、`arxiv_translate`、`arxiv_download`、`arxiv_src`、`parse_arxiv_command`，
LLM 可主动调用查询/翻译/下载，也可代为执行斜杠命令。

## 用法示例

```
@bot /arxiv search large language model
@bot /arxiv search au:vaswani AND ti:attention -t
@bot /arxiv get 1706.03762 -t
@bot /arxiv tr 1706.03762
@bot /arxiv dl 1706.03762
@bot /arxiv dl 1706.03762 2105.02723
@bot /arxiv src 1706.03762
```

群聊需先 @ 机器人；私聊无需 @。斜杠命令白名单留空 = 所有用户可用。

## 配置（插件设置页）

- `max_results`：默认搜索条数（1-20，默认 5）
- `request_timeout`：API 请求超时秒数（默认 15）
- `sort_by`：默认排序（relevance / submittedDate / lastUpdatedDate）
- `download_dir`：PDF 保存目录（默认 `data/files/arxiv_pdf`）
- `source_dir`：LaTeX 源码保存目录（默认 `data/files/arxiv_src`）
- `command_prefix`：斜杠命令前缀（默认 `/arxiv`）
- `enable_commands`：斜杠命令总开关
- `slash_whitelist`：斜杠命令白名单 QQ 列表
- `translate_enabled`：是否启用 LLM 翻译（默认开）
- `translate_lang`：翻译目标语言（zh/en/ja/ko，默认 zh）

## 技术要点

- 解析 Atom XML：标题/摘要/作者/发布日期/更新日期/分类/PDF 链接（`link[title=pdf]`）
- 遵守 arXiv API 礼貌间隔（两次请求 >= 3 秒），模块级锁 + 时间戳节流
- PDF / 源码并发下载（信号量上限 3），临时文件 + `os.replace` 原子落盘
- PDF 校验 `%PDF` 魔数；源码校验非空且非 HTML 错误页，扩展名按 Content-Type / 魔数推断
- 结果 TTL 缓存（10 分钟），减少重复 API 调用
- arXiv ID 白名单正则校验（`\d{4}\.\d{4,5}` 或 `cat/\d{7}`，可带 `vN`），防止路径穿越
- 搜索支持 arXiv 高级语法：`ti:` / `au:` / `abs:` / `cat:` / `all:` 前缀与 AND/OR/NOT
- 翻译使用默认 LLM（`get_default_llm_client()`），失败友好提示「翻译服务不可用，先试试 /arxiv get」

## 文件清单

```
data/plugins/arxiv_search/
├── manifest.json   # 插件元信息
├── schema.json     # 配置项定义
├── main.py         # 插件主逻辑
├── __init__.py     # 包标记
└── README.md       # 本文件
```

下载的 PDF 存放于 `data/files/arxiv_pdf/`，LaTeX 源码存放于 `data/files/arxiv_src/`
（文件名 = 经 `_sanitize_id` 处理后的 arXiv ID，旧式 ID 的 `/` 替换为 `_`）。