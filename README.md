# MIS_Chat_Bot
**# 🤖 ReportBot — AI Report Chatbot for MySQL (Telegram + Web)

Ask questions about your MySQL data in **English, हिंदी, ગુજરાતી or मराठी**, and get answers as a **chat table, Excel, PDF, chart or HTML report**.
Teach the bot your company's report formats once (rules, layout, cross-checks), then run them any time as `/commands`.

Powered by **Claude (Anthropic)** · Python · FastAPI · Docker

---

## ✨ Features

| Feature | Description |
|---|---|
| 💬 Multilingual chat | English, Hindi, Gujarati, Marathi, including Hinglish ("kal ki sales batao") |
| 📱 Telegram + 🌐 Web | Telegram bot, plus a personal web chat link for every user |
| 🎓 Teachable reports | `/teach` → the bot interviews you, writes and tests SQL, saves the report as `/report_name` |
| 📏 Rules & cross-checks | Business rules and validation checks are stored with each report and followed every run |
| 📊 5 output formats | Chat table, Excel (₹ lakh/crore, totals), PDF (Indic fonts), chart PNG, HTML |
| 🗂 Report history | Every report saved; re-send, compare, or convert old reports |
| ⏰ Scheduled reports | "roj subah 9 baje /daily_sales group me bhejo" |
| 🔐 Role-based access | `admin` / `manager` / `viewer`, per-table and per-report permissions |
| 🛡 Read-only & safe | SELECT-only SQL guard, read-only DB user, query timeout |
| 🔗 n8n workflow | Importable workflow with a Telegram flow and a "Teach Report" form node |

---

## 🏗 How it works

```
Telegram / Web / n8n ──► ReportBot (FastAPI, Docker) ──► Claude API
                               │
                               ├──► MySQL ERP database (read-only)
                               └──► Bot database: users, roles, reports, history, schedules
                                         │
                                         └──► Table · Excel · PDF · Chart · HTML
```

---

## 📋 Prerequisites

- Windows 10/11, Linux or macOS, with **Docker Desktop**
- **MySQL** 5.7 / 8.x (a replica database is recommended)
- **Claude API key**: [console.anthropic.com](https://console.anthropic.com) (starts with `sk-ant-api03-`)
- **Telegram bot token**: from [@BotFather](https://t.me/BotFather)
- Your **Telegram ID**: from [@userinfobot](https://t.me/userinfobot)
- **cloudflared**: for a public HTTPS link ([download](https://github.com/cloudflare/cloudflared/releases/latest))

---

## 🚀 Quick start

### 1. Clone
```bash
git clone https://github.com/<your-username>/reportbot.git
cd reportbot
```

### 2. Create the MySQL user (run as a MySQL admin)
```sql
CREATE DATABASE IF NOT EXISTS reportbot CHARACTER SET utf8mb4;

CREATE USER 'reportbot'@'%'         IDENTIFIED BY '<BOT_DB_PASSWORD>';
CREATE USER 'reportbot'@'localhost' IDENTIFIED BY '<BOT_DB_PASSWORD>';

GRANT SELECT ON <ERP_DB_NAME>.* TO 'reportbot'@'%';
GRANT SELECT ON <ERP_DB_NAME>.* TO 'reportbot'@'localhost';
GRANT ALL    ON reportbot.*     TO 'reportbot'@'%';
GRANT ALL    ON reportbot.*     TO 'reportbot'@'localhost';
FLUSH PRIVILEGES;
```
> Use letters and numbers only in the password. Symbols like `@` or `#` break the connection URL.

### 3. Configure
```bash
cp .env.example .env      # Windows: copy .env.example .env
```
Then edit `.env`:
```env
ANTHROPIC_API_KEY=<YOUR_CLAUDE_KEY>
DATA_DB_URL=mysql+pymysql://reportbot:<BOT_DB_PASSWORD>@host.docker.internal:3306/<ERP_DB_NAME>?charset=utf8mb4
BOT_DB_URL=mysql+pymysql://reportbot:<BOT_DB_PASSWORD>@host.docker.internal:3306/reportbot?charset=utf8mb4
TELEGRAM_BOT_TOKEN=<BOTFATHER_TOKEN>
PUBLIC_BASE_URL=http://localhost:8000
BOOTSTRAP_ADMIN_TELEGRAM_ID=<YOUR_NUMERIC_TELEGRAM_ID>
BOOTSTRAP_ADMIN_WEB_TOKEN=<ADMIN_WEB_PASSWORD>
COMPANY_NAME=<COMPANY_NAME>
```
`host.docker.internal` means MySQL is on the same machine. Otherwise, use the MySQL server's IP.

### 4. Build & start
```bash
docker compose up -d --build
docker compose exec reportbot python -m tools.manage init     # prints the admin web link
```
Open `http://localhost:8000/?token=<ADMIN_WEB_PASSWORD>`.

### 5. Public link + Telegram
```bash
cloudflared tunnel --url http://localhost:8000
```
Copy the `https://xxxx.trycloudflare.com` link into `PUBLIC_BASE_URL` in `.env`, then:
```bash
docker compose up -d --force-recreate
docker compose exec reportbot python -m tools.manage set-webhook
```
Open your bot in Telegram and send `/start`.

---

## 👥 Users & roles

| Role | Permissions |
|---|---|
| `admin` | Everything, including managing users |
| `manager` | All data, free questions, teach reports, schedules |
| `viewer` | Run already-taught reports only |

```bash
docker compose exec reportbot python -m tools.manage add-user "Name" viewer            # web user
docker compose exec reportbot python -m tools.manage add-user "Name" manager 123456789  # + Telegram
docker compose exec reportbot python -m tools.manage users                             # list users
```
In Telegram, admins can also use: `/adduser <telegram_id> <role> <Name>`

Deactivate a user:
```sql
UPDATE reportbot.rb_users SET active = 0 WHERE id = <ID>;
```

---

## 🎓 Teaching a report

```
/teach
Report name: daily_dispatch
FORMAT:  Branch | Challan Count | Qty (MT) | Amount (₹), sorted by Amount, TOTAL row, Excel + PDF + bar chart
RULES:   1. Use challan_date  2. Exclude cancelled (status='C')  3. Amount without GST  4. Default: yesterday
CROSS-CHECKS: 1. TOTAL Qty = total from dispatch table  2. Challan count = distinct challans  3. No blank branch
```
The bot asks questions, shows a preview, and saves the report when you say **"save"**. Then:

| Command | Result |
|---|---|
| `/daily_dispatch` | Yesterday's report |
| `/daily_dispatch Pune 1-15 Oct excel` | Custom filter / format |
| `/skills` · `/history` · `/schedules` | List reports / history / schedules |
| `/teach daily_dispatch` | Edit an existing report (versions are kept) |

You can also attach an existing Excel or PDF report with the caption *"is format me banao"* and the bot copies its layout.

---

## 🛠 Useful commands

| Command | Purpose |
|---|---|
| `docker compose ps` | Status (should be **Up**) |
| `docker compose logs --tail 30 reportbot` | Recent logs / errors |
| `docker compose up -d --force-recreate` | Apply `.env` changes |
| `docker compose up -d --build` | Apply code changes |
| `docker compose down` | Stop |
| `python -m pytest -q tests/` | Run tests (no keys needed) |

---

## 🧯 Troubleshooting

| Problem | Fix |
|---|---|
| `docker` not recognized (Windows) | Add `%LOCALAPPDATA%\Programs\DockerDesktop\resources\bin` to Path |
| Container keeps restarting | `docker compose logs --tail 40 reportbot` |
| `Access denied for user 'root'@'localhost'` | Use the dedicated `reportbot` MySQL user |
| `Unknown database 'reportbot'` | `CREATE DATABASE reportbot CHARACTER SET utf8mb4;` |
| `set-webhook: Failed to resolve host` | Tunnel not running, or `PUBLIC_BASE_URL` still localhost |
| Telegram: "You are not registered" | `UPDATE reportbot.rb_users SET telegram_id='<ID>' WHERE id=1;` |
| "AI service error" | Check that the Claude key starts with `sk-ant-api03-` and billing is active |

---

## 📁 Project structure

```
app/
  agent.py        Claude agent: tools, prompts, teaching interview
  datadb.py       Read-only MySQL access + SQL safety guard
  store.py        Users, roles, report skills, history, schedules
  renderers.py    Table / Excel / PDF / chart / HTML output
  telegram.py     Telegram webhook, commands, file delivery
  scheduler.py    Scheduled reports
  main.py         FastAPI routes
  static/         Web chat UI
n8n/              Importable n8n workflow
sql/              MySQL schema for bot tables
examples/skills/  Sample report definitions
tools/manage.py   Admin CLI
tests/            End-to-end tests
```

---

## 🔒 Security

- **Never commit `.env`.** It is excluded by `.gitignore`.
- Always use a **read-only** database user, never `root`.
- Each web link is personal; rotate a user's `web_token` if it is shared.
- If a key or token leaks: revoke it in @BotFather (`/revoke`) or the Anthropic console.

---

## 📄 License

MIT. Free to use and modify.**
