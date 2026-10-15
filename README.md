# 班级日历助手（class-calendar-bot）

把 WeLink 班级群里老师、班委发的通知，**自动抽成日历上的日程**，并在每天早上把当天/本周安排推回群里。再也不用往上翻 200 条聊天记录找「下周一交什么作业」。

```
09:12  李老师：下周一前交高数作业，纸质版交给学委
09:12  王班长：明天下午2点在A301开班会，请各位准时到
09:13  学习委员：10月20日上午9:00-11:00 英语四级考试，地点：教学楼B201

        ↓  Bot 自动解析入库

08:00  【班级日程】未来 3 天
       【作业】
       📚 · [周一 23:59 截止] 高数作业
       【会议】
       🧑‍🏫 · [明天 14:00-15:00] 班会 @A301
       【考试】
       📝 · [周二 09:00-11:00] 英语四级考试 @教学楼B201
```

同时导出 `class-calendar.ics`，手机点一下就能订阅进系统日历。

---

## 目录结构

```
class-calendar-bot/
├── main.py                # 入口：HTTP 服务 + 定时推送 + 命令行
├── parser.py              # 核心：中文通知 -> 结构化日程
├── store.py               # 事件存储（JSON，原子写入）
├── welink.py              # WeLink 回调验签/解密 + 机器人推送
├── ics_export.py          # 导出 .ics + 生成群摘要文本
├── test_parser.py         # 35 个单元测试
├── config.example.json    # 配置模板（复制成 config.json 使用）
├── tasks.example.json     # 数据格式示例
├── messages.example.json  # 示例群消息，可直接导入体验
├── requirements.txt
└── .gitignore             # 已排除 config.json / tasks.json / *.ics
```

---

## 快速开始

```bash
# 1. 装依赖
python -m venv .venv
.venv\Scripts\activate            # Windows；macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt

# 2. 准备配置
cp config.example.json config.json   # Windows: copy config.example.json config.json
#    编辑 config.json，填 WeLink 的 webhook_url 等

# 3. 先干跑一遍，不碰网络
python main.py import-json messages.example.json
python main.py digest 7
python main.py ics

# 4. 启动服务
python main.py serve
```

跑测试：

```bash
python -m unittest -v test_parser
```

---

## 配置说明（config.json）

| 字段 | 说明 |
| --- | --- |
| `welink.webhook_url` | 群机器人 Webhook 地址，**推送**消息用 |
| `welink.robot_id` / `app_id` / `app_secret` | WeLink 应用凭证 |
| `welink.callback_token` | 回调签名校验 token |
| `welink.callback_aes_key` | 43 位 EncodingAESKey，**接收**加密消息用 |
| `welink.verify_callback_signature` | 是否校验回调签名，生产环境务必 `true` |
| `server.host` / `port` / `path` | HTTP 服务监听地址与回调路径 |
| `push.enabled` | 是否开启定时推送 |
| `push.times` | 推送时刻，如 `["08:00", "21:30"]` |
| `push.digest_days` | 摘要覆盖未来多少天 |
| `push.notify_on_new_event` | 收到新日程时是否在群里回执一条"已记录" |
| `calendar.ics_output` | 导出的 .ics 文件名 |
| `store.path` | 数据文件路径 |
| `store.purge_after_days` | 超过多少天的历史事件会被清理 |

> `config.json` 和 `tasks.json` 已在 `.gitignore` 里，**不会**被提交。仓库里只放 `*.example.json`。

---

## WeLink 接入

**推送（必做）**：群设置 → 群机器人 → 添加机器人 → 拿到 Webhook 地址 → 填到 `welink.webhook_url`。这一步就有了「往群里发消息」的能力。

**接收（可选）**：在 WeLink 开放平台建应用，把回调地址指向 `http://<公网地址>:8080/welink/callback`，填好 `callback_token` 与 `callback_aes_key`。
解密需要额外依赖：`pip install cryptography`（不装则自动退化为明文模式，仅适合本地联调）。

没有公网服务器的话，可以用内网穿透工具把本机 8080 暴露出去，或者干脆只用命令行 `import-json` 手动喂消息。

---

## 命令行

```bash
python main.py serve                            # 启动服务（默认）
python main.py parse "明天下午2点在A301开班会"     # 试解析，不写库
python main.py import-json messages.json        # 批量导入历史消息
python main.py digest 3                         # 打印未来 3 天摘要
python main.py digest 3 --push                  # 打印并推送到群
python main.py ics [输出路径]                    # 导出 .ics
python main.py stats                            # 存储统计
```

## HTTP 接口

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/health` | 健康检查，返回事件总数与分类统计 |
| GET | `/welink/callback?echostr=xxx` | 回调地址验证，原样回显 echostr |
| POST | `/welink/callback` | 接收群消息（支持加密与明文两种模式） |
| POST | `/parse` | 调试用：`{"text": "..."}` 返回解析结果 |
| GET | `/events?days=30` | 未来 N 天的日程列表 |
| GET | `/calendar.ics` | 下载 .ics 日历文件 |

---

## 支持哪些写法

**日期**：`今天 / 明天 / 后天 / 大后天 / 今晚 / 明晚`、`周三 / 下周三 / 本周五 / 下下周`、`10月20日`、`2026-10-20`、`2026年10月20日`

**时刻**：`9点 / 上午9:30 / 下午2点半 / 晚上7点 / 两点半 / 9:00-11:00 / 9点到11点`

**类型**：考试 📝 / 作业 📚 / 会议 🧑‍🏫 / 活动 🎉

**地点**：`在A301`、`地点：教学楼B201`、`图书馆报告厅`、`体育馆`

**截止型**：`下周一前交高数作业` → 自动落到当天 23:59 并标记 `[截止]`

### 有意为之的取舍

- **闲聊不进日历**。一条消息必须至少满足其中之一才收：有明确时刻、能判出类型、含截止词、或含地点。
  所以「明天见」「今天天气不错」会被静默丢弃。
- **标题只取第一个分句**。`明天下午2点开班会，请各位准时到` → 标题 `班会`，逗号后的附注不进日历。
- **类型判断在地点剥离之后进行**。否则「图书馆报告厅」里的"报告"会把一场讲座误判成作业。
- **1~6 点没写时段时按下午算**（校园场景），并把 `time_assumed` 标为 `true`，需要时可按此复核。
- **多行消息按行拆分**，一次通知里写了周末安排也能全部收下。

### 已知限制

- 纯规则实现，不理解语义。反讽、条件句、玩笑话可能被误收。
- `报告`/`交` 这类短词偶有歧义（已通过「先剥离地点和日期再判类型」缓解大部分）。
- 没有实现放假/调休日历，跨节假日的相对日期需要人工确认。
- 类型关键词表在 `parser.py` 的 `TYPE_RULES`，是主要调优入口，按学校实际用语补充即可。

---

## 安全

- `config.json` 含 Webhook 密钥，`tasks.json` 含班级真实数据，二者均在 `.gitignore` 中。
- 生产环境请保持 `verify_callback_signature: true`，签名比较使用常量时间实现。
- 回调处理对签名失败返回 403，对解析失败返回 400，不会把内部异常抛给调用方。

## License

MIT
