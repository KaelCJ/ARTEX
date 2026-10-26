# AGENTS.md — 跨机器工作续接指南

> 写给在不同电脑上继续开发本项目的开发者 / AI agent。**每次暂停工作前更新「当前工作状态」一节；每次接手先读它。** 架构与部署细节见 [README.md](README.md)，版本历史见 [CHANGELOG.md](CHANGELOG.md)。

## 项目是什么

**ARTEX** —— AI 自主渗透测试系统（Go 后端 + Next.js 前端，前端经 `go:embed` 内嵌为单二进制）。

- 本仓库是 fork：`origin = https://github.com/KaelCJ/ARTEX.git`（**公开仓库**），上游 `upstream = https://github.com/mhtsec/ARTEX.git`
- 定位：最后一个版本的纯源码备份，Docker 部署源已失效，需本地构建运行（README「安装 · 方式四」）
- 当前版本：**0.3.15**（2026-10-07 定版），主分支 `main`
- 协议：AGPL-3.0；仅限本地隔离环境学习研究，禁止实际攻击

## 当前工作状态（2026-10-26 快照）

- HEAD = `5c0dbd4`（docs(readme): 标注本仓库为最后版本纯源码备份），已与 origin/main 同步
- **本机未提交内容**（换电脑时需从旧机器拷贝或重建）：
  - 新增 `start-dev.cmd`、`start-background.ps1` —— Windows 本地启动脚本（git 未跟踪）
  - `.gitattributes` 加了 `*.cmd text eol=crlf`；`.gitignore` 加了 `/.logs/`
- 本机环境：Windows、Go 1.26.3、Node v24.21.0、PostgreSQL 16（本机服务）、`web/node_modules` 已安装

## 数据库（PostgreSQL）

本机开发库连接信息（同时记录在本机 `config.json`，该文件被 .gitignore 忽略）：

| 项 | 值 |
| --- | --- |
| host | `127.0.0.1` |
| port | `5432` |
| user | `postgres` |
| password | `kkkk1111` |
| dbname | `artex` |
| sslmode | `disable` |

等价 DSN：`postgres://postgres:kkkk1111@127.0.0.1:5432/artex?sslmode=disable`

- 配置优先级：环境变量 `ARTEX_PG_DSN` > `config.json` 的 `database` 字段（分字段或完整 DSN 两种写法，见 `config.example.json`）
- **无需手动建表/迁移**：`artex` 每次启动幂等重跑 `schema.sql`（含 `ADD COLUMN` / `CREATE INDEX IF NOT EXISTS`），重启即迁移
- 新电脑上：装好 Postgres → 建库 `artex` → 复制 `config.example.json` 为 `config.json` 填连接 → 启动自动建表

## 新电脑续接 checklist

1. 装依赖：Go（本机用 1.26.3）、Node.js（本机用 v24）、PostgreSQL 16
2. 数据库：见上一节；或 Docker 方式 `cp .env.example .env` 后 `docker compose up -d`（仅起 postgres 时只启动其 `postgres` 服务）
3. 前端依赖：`cd web && npm ci`
4. 启动（见下节）
5. 打开 http://localhost:5173（dev 前端）或 http://localhost:8787（内嵌模式）；首次进入 `/setup` 设置管理员密码

## 启动与常用命令

Linux/macOS：

```bash
./dev.sh          # 后端(:8787) + 流量代理(:8788) + 前端 next dev(:5173) → http://localhost:5173
go test ./...     # 后端测试
```

Windows（本机脚本，⚠️ 尚未提交到 git，见「当前工作状态」）：

- `start-dev.cmd` —— 起后端(8787/8788) + 前端(5173)，带健康检查、自动复用已运行实例
- `start-background.ps1` —— 后台无窗口启动（互斥防重复起，日志在 `.logs/`）；`.claude/launch.json` 引用它（port 5173）
- ⚠️ 两个脚本写死了本机路径：`D:\Development\Go\1.26.3\bin\go.exe`、`D:\Development\NodeJS`、`GOPROXY=goproxy.cn,direct`，**换机必须改**；`.claude/launch.json` 里的绝对路径同理

手动启动（任何平台）：

```bash
go run ./cmd/artex -addr 127.0.0.1:8787 -proxy 127.0.0.1:8788   # 后端 + 流量代理
cd web && npm run dev                                           # 前端热更新（/api 反代到 8787）
cd web && NEXT_PUBLIC_MOCK=1 npm run dev                        # 无后端 Mock 预览
```

构建发布：`./build.sh --release`（前端 `npm run build:static` → 拷 `web/out` 到 `server/webui/dist` → `go build -tags embedui`）

## 架构速览（详见 README「系统技术架构」）

- **双图架构**：资产图（全局共享真值库：root_domain/subdomain/ip/service/app/endpoint）+ 探索图（每任务独立：goal/intent/fact/finding/hint），靠 `exploration_anchors` 锚点相连，支撑资产测试覆盖图
- **事件驱动闭环**：图变更 → 唤醒 planner → 派意图进 frontier → worker 领取执行、写回两图 → 再次唤醒，直到 `prove_goal`
- planner 持有跨唤醒共享 todolist，稳定推进多步利用链；worker 间通过 `search_all_worker_traces` / `get_worker_trace` 共享执行过程
- 关键目录：`agent/`（goals/planner/worker/mainagent）、`db/`（双图 Postgres 落地，pgx）、`server/`（net/http + JWT + SSE）、`cmd/artex/`、`web/`（Next.js）
- 支撑子系统：MITM 流量记录代理(8788)、guard/intercept 审批门、DNS/HTTP 异步补全、MCP/skills/memory/report

## 配置要点

- `config.json`：仅 `database` 与 `skill_dir` 两项；LLM 等其余配置走环境变量或 UI「LLM 配置」页
- LLM：`ANTHROPIC_API_KEY` / `OPENAI_API_KEY`；可选 `ARTEX_LLM_PROVIDER` / `ARTEX_LLM_MODEL` / `ARTEX_LLM_BASE_URL` / `ARTEX_LLM_PROXY`
- 端口：8787 = 前端+API（内嵌模式），8788 = 流量录制代理，5173 = next dev
- gitignored 但重要的本机文件：`config.json`（数据库密码）、`.env`、`data/`（jwt.key、SQLite）、`jwt.key`、`.logs/`、`.claude/`

## 坑与注意

- 发布产物别直接跑 `./artex`，要用 `start.sh` / `start.bat`（守护换装，页面一键更新依赖它）
- `docs/` 在 .gitignore 里，但 `docs/漏洞流量证据.md` 已被跟踪、可正常提交
- 升级只换程序不动数据（`pgdata` 卷、`./data`、`./skills` 保留）；数据库结构不随回滚回退
- 远程 MCP：系统设置里选 `http`（Streamable HTTP）或 `sse`（URL 填 `/sse`，头 `Authorization=Bearer <token>`）
