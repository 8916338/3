# 海量接口搬运备份站

一个基于 GitHub Actions 自动化 + 静态托管的 TVBox 接口搬运备份项目。三套脚本每日（北京时间）定时抓取、解析、聚合各类 TVBox 接口与直播源，产物自动提交回仓库，前端页面实时读取展示，支持一键复制、搜索、多域名备份。

---

## 目录

- [项目简介](#项目简介)
- [在线访问](#在线访问)
- [仓库结构](#仓库结构)
- [自动化工作流](#自动化工作流)
- [脚本与工具](#脚本与工具)
- [时间预算与并发策略](#时间预算与并发策略)
- [前端页面](#前端页面)
- [配置文件说明](#配置文件说明)
- [本地手动运行](#本地手动运行)
- [部署到 Cloudflare Pages](#部署到-cloudflare-pages)
- [常见问题与避坑](#常见问题与避坑)
- [配置修改速查](#配置修改速查)
- [一键校验清单](#一键校验清单)
- [声明](#声明)

---

## 项目简介

本项目把互联网上公开的 TVBox 接口资源进行批量搬运、备份与聚合，通过 GitHub Actions 定时抓取并提交到仓库，再由静态托管前端页面，实现：

- **接口备份**：`api_list.json` 中登记的每条点播接口可配多镜像线路，脚本自动选通可用镜像。
- **直播聚合**：从所有点播接口的 `lives` 字段提取直播源，穿透套壳后生成播放列表。
- **采集站聚合**：从所有接口的 `sites` 中提取 type 0/1 的采集站，生成独立聚合接口。
- **Py 站聚合**：提取以 `.py` 结尾的 Py 站接口，补全字段后生成聚合接口。
- **下载资源**：批量抓取本地包/下载资源，落到 `ry/` 目录。
- **多域名备份**：前端提供三个互为备份的访问域名，任一可用。
- **静态托管**：纯前端页面，无需后端，部署简单，访问快。
- **稳**：并发抓取 + 硬性时间预算，单个源卡死不会拖垮整轮任务；失败时保留上一次的产物。

> 所有时间（`list.txt`、`livelist.txt` 的日期列，`SUMMARY.txt` 的表头时间）统一使用**北京时间（UTC+8）**，不受 GitHub runner 默认 UTC 影响。

---

## 在线访问

| 域名 | 用途 |
|------|------|
| https://lubin776.github.io/tvbox-api-backup/ | GitHub Pages 主站 |
| https://0.12yue.de5.net | 备份一线 |
| https://0.cdz.qzz.io | 备份二线 |
| https://0.wdzb.eu.cc | 备份三线 |

域名指向同一份静态站点，互为备份。

---

## 仓库结构

```
tvbox-api-backup/
├── .github/workflows/
│   ├── ry下载.yml                 工作流A：每日解析本地包（北京 01:00）
│   ├── api下载.yml                工作流B：每日抓取点播接口（北京 00:05）
│   ├── 接口处理扫描工具.yml        工作流C：每日直播/采集/Py 聚合（北京 03:00）
│   └── Download.yml               手动：从固定 zip 解压覆盖到仓库根
│
├── python/                        脚本目录
│   ├── fetch_util.py              公共抓取层（时间预算 / 连接池 / 主机熔断）
│   ├── tvbox_get_api.py           脚本B：点播接口抓取（工作流B 调用）
│   ├── 接口扫描处理工具.py          脚本C：直播+采集+Py 聚合（工作流C 调用）
│   ├── ry下载器.py                 脚本A：本地包解析（工作流A 调用）
│   ├── live下载器.py               仅直播的独立下载脚本（未接入工作流）
│   ├── api下载器.py                点播抓取旧版（未接入工作流）
│   ├── api下载器2.py               点播抓取旧版，读 apilinks.txt（未接入工作流）
│   └── 接口本地化工具.py            接口本地化下载工具（未接入工作流）
│
├── api_list.json                  脚本B 配置（优先读取）
├── api_list.py                    脚本B 配置（json 不存在时兜底）
├── rylinks.txt                    脚本A 配置（名称,URL）
├── apilinks.txt                   旧版点播配置（仅 api下载器2.py 使用）
│
├── ry/                            脚本A 产物
├── tvbox/                         脚本B/C 产物
│   ├── <接口名>.json              各线路接口文件
│   ├── 海量直播聚合接口.json        脚本C 聚合锚点
│   ├── 海量采集聚合接口.json        脚本C 聚合锚点
│   ├── 海量py聚合接口.json          脚本C 聚合锚点
│   └── live/                      脚本C 播放列表输出（.m3u / .txt）
│
├── list.txt                       脚本B 点播接口清单（日期=北京时间）
├── SUMMARY.txt                    脚本B 运行摘要
├── livelist.txt                   脚本C 直播源索引（日期=北京时间）
├── cjlist.txt                     脚本C 采集站列表
├── pylist.txt                     脚本C Py 站列表
│
├── index.html                     前端主页面（单文件，样式与脚本内联）
├── app.html / a.html              备用页面
├── firstpage.html / into.html     引导页
├── qq.html                        联系页
├── config.json                    前端内容配置（下载源 + 展示内容）
├── CNAME                          GitHub Pages 自定义域名
├── cy/                            书源/音源/漫画源等扩展配置
├── vip/                           前端图片资源
└── README.md / 项目说明.md         文档
```

---

## 自动化工作流

三个定时工作流位于 `.github/workflows/`，推送到默认分支后 GitHub 自动识别。三者共用 `concurrency: tvbox-repo-write`（`cancel-in-progress: false`），**排队串行执行**，避免并发 push 冲突。

| 工作流 | Cron（UTC） | 北京时间 | 频率 | 执行的脚本 | 产物 |
|--------|-------------|---------|------|-----------|------|
| `ry下载.yml` | `0 17 * * *` | 每日 01:00 | **每天 1 次** | `python/ry下载器.py` | `ry/` |
| `api下载.yml` | `5 16 * * *` | 每日 00:05 | **每天 1 次** | `python/tvbox_get_api.py` | `tvbox/*.json`、`list.txt`、`SUMMARY.txt` |
| `接口处理扫描工具.yml` | `0 19 * * *` | 每日 03:00 | **每天 1 次** | `python/接口扫描处理工具.py --force` | `tvbox/live/`、三个聚合 JSON、`livelist.txt`、`cjlist.txt`、`pylist.txt` |
| `Download.yml` | 无 | — | 仅手动触发 | —（下载 zip 并解压） | 覆盖仓库根文件 |

### 工作流特性

- **每天一次**：三个定时任务均为单条 `cron`，每日触发一次，没有「奇偶日」「多时段」之类的额外节流。
- **额外触发**：`接口处理扫描工具.yml` 额外监听 `push`（仅 `python/**` 与该 yml 自身），改脚本后会立即跑一次验证；`tvbox/**` 是产物目录，不再触发，避免产物提交引发重复运行。
- **手动触发**：均支持 `workflow_dispatch`，可勾选 `debug` 输出详细日志。
- **超时保护**：`api下载.yml` 设 `timeout-minutes: 30`，`接口处理扫描工具.yml` 设 `timeout-minutes: 60`；脚本内部还有更短的预算，正常情况下远用不到。
- **依赖安装**：`pip install requests urllib3`（仓库未提供 requirements.txt）。
- **提交产物**：
  - 工作流A：`git add ry/`
  - 工作流B：`git add tvbox/ list.txt SUMMARY.txt`
  - 工作流C：`git add -A`（产物分布在 `tvbox/live/` 与仓库根多个文件）
  - 提交前均先 `git pull --rebase`（A、B 带 `--autostash`）。

---

## 脚本与工具

### 脚本 A：`python/ry下载器.py`

**作用**：批量解析「本地包/下载资源」类接口，把结果落到 `ry/` 目录，并生成 `ry/_summary.json`。

**配置：`rylinks.txt`（仓库根）**

```
潇洒下载, https://9877.kstore.space/single.json
奇奇下载, http://bd.qiqiv.cn/666.json
```

- 每行 `名称, URL`（英文逗号，两边可有空格），`#` 开头为注释，名称可省略
- 支持 `file://`（本地）和 `raw:base64`（内联）用于调试

**运行方式**

```bash
python3 python/ry下载器.py
python3 python/ry下载器.py -c /path/to/other.txt   # 指定配置
python3 python/ry下载器.py -l "URL" -n 测试         # 单链接调试
python3 python/ry下载器.py --waf                   # 本地 mock 模式
python3 python/ry下载器.py --debug
```

---

### 脚本 B：`python/tvbox_get_api.py`（点播）

**作用**：抓取 TVBox 点播接口（`api_list.json` 中登记的条目，每条可带多镜像），逐个解密/解析/补全相对路径，生成 `tvbox/<接口名>.json` + `list.txt` + `SUMMARY.txt`。

**配置加载优先级**

| 优先级 | 文件 | 说明 |
|--------|------|------|
| 1 | `api_list.json` | 存在即用（当前仓库使用） |
| 2 | `api_list.py` | json 不存在时作为 Python 模块导入 |

`api_list.json` 格式：

```json
{
  "API_LIST": [
    ["X4更新专用接口", "https://0.12yue.de5.net/0/666"],
    ["饭太硬", "http://www.饭太硬.net/tv"]
  ],
  "API_MIRRORS": {
    "摸鱼": ["http://www.y456y.com", "http://fish.y456y.com"]
  }
}
```

同名条目的 URL 会与 `API_MIRRORS` 合并去重，按名称分组后逐个尝试。

**抓取策略（本版重点）**

| 机制 | 说明 |
|------|------|
| 单镜像独立预算 | 每个 URL 最多 12 秒，失败立刻换下一个镜像，坏镜像不再吃光全局时间 |
| UA 重试收敛 | 每个地址最多换 2 个 UA；单次耗时超过 9.6 秒就不再换，避免时间翻倍 |
| 上次成功地址优先 | 从 `list.txt` 读回上次成功的 URL，排到镜像列表首位，成功率更高 |
| 接口级并发 | 默认 6 路（`VOD_WORKERS`） |
| 整轮预算 | 默认 20 分钟（`VOD_BUDGET`），到点收工，未完成的接口记为 `SKIPPED` 并保留旧文件 |
| 失败补抓 | 首轮失败的接口在剩余预算内再补抓一轮 |

**产物规则**：只有解析成功（`status == JSON`）的接口才写进 `list.txt`；失败/跳过的接口**不会**覆盖旧文件与旧记录，上游临时抽风不会把仓库产物弄坏。

**运行方式**

```bash
python3 python/tvbox_get_api.py                  # 常规抓取
python3 python/tvbox_get_api.py --debug          # 详细日志
python3 python/tvbox_get_api.py --check-config   # 只检查配置分组，不联网
python3 python/tvbox_get_api.py --selftest       # 内置自测
```

---

### 脚本 C：`python/接口扫描处理工具.py`（直播 + 采集 + Py 聚合）

**作用**：TVBox 接口全量扫描与聚合工具，覆盖**直播源下载**、**采集站提取**、**Py 站提取**三大模块，模拟 TVBox 客户端环境，生成播放列表与多种聚合索引。

**核心原则**：输出文件名锚定原始条目名（不受套壳跳转影响），保证对外访问路径长期稳定。

#### 模块一：直播源（lives）

1. **扫描提取**：遍历 `tvbox/*.json`（按文件名排序，结果稳定）→ 读取 `lives` → 提取有 name 且有 url 的条目，URL 去重
2. **聚合命名**：重名自动编号 `xxx → xxx2线 → xxx3线`，写入 `tvbox/海量直播聚合接口.json`
3. **并发下载**：默认 8 路并发（`LIVE_WORKERS`），每个源 20 秒硬预算，套壳最多展开 5 层，输出播放列表到 `tvbox/live/`
4. **生成索引**：更新 `livelist.txt`（第一列为磁盘真实文件名，日期为北京时间；旧记录保留，新记录覆盖）

#### 模块二：采集站（sites type 0/1）

5. **扫描提取**：遍历 `tvbox/*.json` → 读取 `sites` → 提取 `type` 为 0 或 1 的条目（按 api 去重）
6. **输出列表**：写入 `cjlist.txt`，格式 `标识|名称|type: x|api: http...`
7. **生成聚合接口**：转换为标准站点格式后插入模板，输出 `tvbox/海量采集聚合接口.json`

#### 模块三：Py 站（sites api 以 .py 结尾）

8. **扫描提取**：同上，筛选 `api` 以 `.py` 结尾的条目
9. **输出列表**：写入 `pylist.txt`
10. **生成聚合接口**：补全 searchable/quickSearch/filterable 后输出 `tvbox/海量py聚合接口.json`

**运行方式**

```bash
python3 python/接口扫描处理工具.py
python3 python/接口扫描处理工具.py --debug
python3 python/接口扫描处理工具.py --force
```

---

### 其他脚本

| 脚本 | 用途 | 接入工作流 |
|------|------|-----------|
| `python/live下载器.py` | 只做直播源扫描与下载（不含采集站/Py 站），逻辑与脚本C 模块一相同，便于单独调试 | ❌ |
| `python/fetch_util.py` | 公共抓取层：单 URL 硬性时间预算、线程级连接池、失败主机熔断。被直播与点播脚本共同引用 | 依赖库 |
| `python/api下载器.py` | 点播抓取旧版（读 `api_list.json`） | ❌ |
| `python/api下载器2.py` | 点播抓取旧版，读 `apilinks.txt`（新 txt 格式），支持 `--config` | ❌ |
| `python/接口本地化工具.py` | 抓取 + 解密 + 规范化后做本地化下载 | ❌ |

> `apilinks.txt` 仅 `api下载器2.py` 使用；当前工作流跑的是 `tvbox_get_api.py`，配置入口是 `api_list.json`。

---

## 时间预算与并发策略

所有网络抓取都通过 `python/fetch_util.py`，统一了三件事：**单次请求的总时长上限**（不是 requests 那个只管单次 socket 读的 `timeout`）、**连接复用**、**失败主机熔断**。

| 参数 | 环境变量 | 默认 | 说明 |
|------|----------|------|------|
| 直播并发 | `LIVE_WORKERS` | 8 | 同时下载的直播源数量 |
| 直播总预算 | `LIVE_BUDGET` | 1500s | 到点收工，剩余源跳过并保留旧产物 |
| 点播并发 | `VOD_WORKERS` | 6 | 同时抓取的接口数量 |
| 点播总预算 | `VOD_BUDGET` | 1200s | 同上 |

脚本内常量（改代码后生效）：`REQUEST_TIMEOUT`（点播单镜像 12s）、`DOWNLOAD_TIMEOUT`（直播单源 20s）、`PROCESS_BUDGET`（单接口 60s）、`CONNECT_TIMEOUT` / `READ_TIMEOUT`（5s / 10s，位于 fetch_util）。

**行为约定**：无论超预算还是抓取失败，都**不会**删除或清空已有产物，也不会让工作流因异常而整体判失败——最坏情况是「这一轮某些条目没更新」。

---

## 前端页面

前端为纯静态页面，主页面是**单文件** `index.html`（样式与脚本内联，仓库内没有单独的 `style.css` / `js/` 目录），内容配置来自 `config.json`，部署在静态托管平台。

### 页面结构

页面顶部提供模式切换按钮，右上角是搜索框，下方是导航标签（点播 / 直播 / 下载 / 关于），主体区根据标签展示对应面板。

**1. 点播面板**

读取 `list.txt`，每条接口展示：
- 原始线路：接口的原生 URL
- 备份一线 / 备份二线 / 备份三线：基于三个备份域名拼出的 URL

每条 URL 右侧提供「复制」按钮，一键复制到剪贴板。

**2. 直播面板**

顶部是「海量直播聚合多线路专用接口」聚合块，提供聚合接口的复制按钮。

下方读取 `livelist.txt`，每条直播源展示来源、UA、原始线路与三个备份线路。

**3. 采集站面板**

读取 `cjlist.txt`，展示所有提取到的采集站（标识、名称、type、api 地址）。

**4. Py 站面板**

读取 `pylist.txt`，展示所有提取到的 Py 站接口。

**5. 下载面板**

读取 `config.json` 中配置的 `downloadSources`，逐个拉取并渲染下载卡片（图标、名称、版本、跳转链接）。

**6. 关于面板**

展示备份域名、声明与联系方式（QQ 群、Telegram 群）。

### 模式切换

| 模式 | 说明 |
|------|------|
| 简洁模式（默认） | 只显示点播面板，隐藏导航标签与直播/下载/关于面板，搜索框置于顶部 |
| 全能模式 | 显示完整导航标签与所有面板，搜索框置于标签下方 |

模式选择保存在 localStorage，刷新后保持。

### 搜索与复制

- 搜索框对所有面板生效，按卡片 `data-name` 模糊匹配，实时过滤
- 所有 `.copy-btn` 点击后复制 `data-url`，优先 `navigator.clipboard`，降级 `execCommand('copy')`，成功后按钮变绿显示「✅ 已复制」

---

## 配置文件说明

| 文件 | 位置 | 格式 | 归属 | 说明 |
|------|------|------|------|------|
| `api_list.json` | 仓库根 | JSON | 脚本B | 点播接口 + 镜像（优先读取） |
| `api_list.py` | 仓库根 | Python | 脚本B | 点播接口（json 缺失时兜底） |
| `rylinks.txt` | 仓库根 | 文本 | 脚本A | 本地包下载源 |
| `apilinks.txt` | 仓库根 | 文本 | 旧版 `api下载器2.py` | 新 txt 格式接口列表 |
| `config.json` | 仓库根 | JSON | 前端 | 下载资源源 + 内容展示 |

`config.json` 示例：

```json
{
  "downloadSources": [
    {
      "name": "应用市场",
      "description": "TVBox 应用合集",
      "url": "https://cdn.jsdelivr.net/gh/SimonWang911/simonwangsub@main/appupdate/appupdate.json"
    }
  ]
}
```

> 内容展示部分已纳入 `config.json`，自由增减修改无需改动 HTML。

---

## 本地手动运行

```bash
# Step 1: 克隆并进入仓库根
git clone https://github.com/lubin776/tvbox-api-backup.git
cd tvbox-api-backup

# Step 2: 安装依赖（工作流只需这两个）
pip3 install requests urllib3

# Step 3-5: 依次运行三个主脚本
python3 python/ry下载器.py --debug
python3 python/tvbox_get_api.py --debug
python3 python/接口扫描处理工具.py --debug

# Step 6: 验证产物
ls ry/ tvbox/ tvbox/live/ list.txt SUMMARY.txt livelist.txt cjlist.txt pylist.txt
```

注意：脚本靠 **CWD = 仓库根** 定位配置和产物，务必在仓库根执行。

想临时调整并发/预算，可以用环境变量：

```bash
LIVE_WORKERS=16 LIVE_BUDGET=600 python3 python/接口扫描处理工具.py
VOD_WORKERS=10 VOD_BUDGET=600 python3 python/tvbox_get_api.py
```

---

## 部署到 Cloudflare Pages

1. 登录 Cloudflare Dashboard，进入 Pages。
2. 创建项目，关联 GitHub 仓库 `lubin776/tvbox-api-backup`。
3. 构建设置：
   - Framework preset：None
   - Build command：留空
   - Build output directory：`/`（仓库根）
4. 保存并部署，Cloudflare 会分配一个 `*.pages.dev` 域名。
5. 在 Custom domains 中绑定 `0.12yue.de5.net`、`0.cdz.qzz.io`、`0.wdzb.eu.cc` 三个域名。
6. 每次 GitHub Actions 提交产物后，Cloudflare Pages 会自动重新部署（或配置 Webhook 触发）。

---

## 常见问题与避坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 某个点播接口报「全部失败」 | 所有镜像都不可达 / 被拦 | 看日志里每个镜像各自的报错，`--debug` 看解密链路 |
| 某个接口日期一直不更新 | 它已连续多轮失败，`list.txt` 保留旧记录是**预期行为** | 修 `api_list.json` 里的镜像地址 |
| 某轮跑了很久 | 有个别源卡死 | 看「超预算跳过」计数；调小 `VOD_BUDGET`/`LIVE_BUDGET`，或加大并发 |
| 日期列看起来「差一天」 | 误按 UTC 理解 | 本仓库日期统一按北京时间计算，自行改脚本请用 `beijing_now()` |
| 套壳未展开 | 返回内容里 URL 多于 1 条 | 正常行为，多条视为播放列表本身 |
| 直播文件名带序号后缀 | 名称冲突自动编号 `xxx2线` | 正常行为，`livelist.txt` 第一列与磁盘文件名一致 |
| livelist 旧记录丢失 | 条目改名导致 stem 匹配不上 | 旧记录按 stem 匹配，大改名会丢失 |
| Actions 页面看不到工作流 | `.yml` 不在 `.github/workflows/` | 检查路径与文件后缀 |
| `requests` ImportError | 依赖未装 | 检查 `pip install requests urllib3` 步骤日志 |
| 读不到点播配置 | 改的是 `apilinks.txt`，但工作流读 `api_list.json` | 改 `api_list.json`（或改用 `api下载器2.py`） |
| 并发 push 冲突 | 三个工作流同时写仓库 | 共享 `concurrency` 锁 + 提交前 `git pull --rebase` |
| YAML 缩进错误 | 缩进不一致 | `uses`/`run` 与 `name` 同级对齐 |

---

## 配置修改速查

| 想做什么 | 改哪里 | 是否需重新部署 |
|---------|--------|---------------|
| 增删 ry 下载链接 | `rylinks.txt` | ❌ 下次定时任务生效 |
| 增删点播接口 / 调整镜像 | `api_list.json` | ❌ 下次定时任务生效 |
| 改点播并发 / 预算 | 工作流 `env` 的 `VOD_WORKERS` / `VOD_BUDGET` | ✅ push 后 |
| 改直播并发 / 预算 | 工作流 `env` 的 `LIVE_WORKERS` / `LIVE_BUDGET` | ✅ push 后 |
| 改调度时间 | 对应 `.yml` 的 `cron`（注意 cron 用 UTC） | ✅ push 后 |
| 改抓取超时细节 | `python/fetch_util.py` 顶部常量 | ✅ push 后 |
| 改产物目录 | 脚本 `OUTPUT_DIR` + 工作流 `git add` | ✅ push 后 |
| 改下载资源源 / 前端内容 | `config.json` | ❌ 自动生效 |
| 改前端样式 | `index.html`（样式内联） | ✅ push 后 |

---

## 一键校验清单

```bash
# 在仓库根执行
echo "=== 1. 结构检查 ==="
ls .github/workflows/ry下载.yml .github/workflows/api下载.yml .github/workflows/接口处理扫描工具.yml
ls python/ry下载器.py python/tvbox_get_api.py python/接口扫描处理工具.py python/fetch_util.py
ls api_list.json rylinks.txt config.json index.html

echo "=== 2. YAML 语法校验 ==="
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ry下载.yml')); print('ry OK')"
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/api下载.yml')); print('api OK')"
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/接口处理扫描工具.yml')); print('scan OK')"

echo "=== 3. 配置校验（不联网） ==="
python3 python/tvbox_get_api.py --check-config

echo "=== 4. 本地试跑 ==="
python3 python/ry下载器.py
python3 python/tvbox_get_api.py
python3 python/接口扫描处理工具.py

ls ry/ tvbox/ tvbox/live/ list.txt SUMMARY.txt livelist.txt cjlist.txt pylist.txt
```

全部通过 → 部署成功。

---

## 声明

本站接口资源由【误道者】整理。所有资源均来自互联网，版权归原作者所有。仅供测试学习使用，请勿用于违法及商业用途，请勿付费购买。如涉及侵权，请联系删除。

- QQ 交流群：1067685939
- Telegram 群组：https://t.me/tg6603

---

🌐 **项目访问地址**：https://lubin776.github.io/tvbox-api-backup/
📦 **仓库地址**：https://github.com/lubin776/tvbox-api-backup


## 更新日志

> 📅 最后更新：2026年11月1日

### v3（2026-11-01）稳定性与文档对齐

| 类型 | 项目 | 说明 |
|------|------|------|
| ✨ 新增 | `python/fetch_util.py` | 统一抓取层：单 URL 总时长预算、线程级连接池、失败主机熔断、超大响应截断 |
| ⚡ 优化 | 直播下载（脚本C、`live下载器.py`） | 串行改为 **8 路并发**；单源预算 25s→20s、重试 3→2 次、失败即换下一个；整轮 25 分钟硬预算。实测全流程从「超 1 小时被强杀」降到 **约 9~21 秒** |
| ⚡ 优化 | 点播抓取（脚本B） | 每个镜像独立预算（不再共享 45s）；UA 重试 6×2→2 次；**上次成功地址优先**；接口级 6 路并发；整轮 20 分钟预算；失败接口补抓一轮。实测成功率 8/9 → **9/9**，耗时 64s → **11s** |
| 🕒 修正 | 时间统一北京时间 | `livelist.txt` 原用 runner 本地时间（UTC），现改为 UTC+8；`list.txt` 同步确认 |
| 🔁 修正 | 调度 | 三个定时任务明确为**每天一次**；`接口处理扫描工具.yml` 的 `push` 触发移除 `tvbox/**`，只监听 `python/**` 与自身 |
| 📝 修正 | 提交范围 | `api下载.yml` 补上 `git add SUMMARY.txt`（此前该文件长期停留在旧版本） |
| 📝 文档 | README / 项目说明 | 按真实文件名、真实 cron、真实配置入口重写；删除「奇偶日 gate」「apilinks.txt 优先」「style.css / js/」等与仓库不符的描述 |

### v2（2025-09-30）

| 项目 | 说明 |
|------|------|
| 脚本C 升级为「接口全量扫描处理工具」 | 覆盖直播源下载、采集站提取、Py 站提取三大模块 |
| 前端内容纳入 `config.json` | 自由增减修改无需改动 HTML |
| 备份域名 | 新增三个互为备份的访问域名 |
| GitHub Pages 主站 | 上线 https://lubin776.github.io/tvbox-api-backup/ |
