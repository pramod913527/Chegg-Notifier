# Chegg-Notifier

Got it ✅ You want to develop a Chegg Question Notification Bot that works 24/7 and doesn’t depend on keeping your laptop/browser open (unlike an extension). Instead, it should run independently (like a backend service or cloud bot), and you want the ability to update cookies (from browser DevTools → network tab) whenever needed.

Let me break it down step by step with a working plan:


---

🔹 High-Level Workflow

1. Login Method

You’ll extract your Chegg session cookie (from browser → Inspect → Network tab → copy request headers).

This cookie will be used by your bot to access Chegg API/pages without logging in every time.



2. Bot Backend (24/7 Running)

Instead of a browser extension, run the bot on:

Cloud Server (VPS / AWS / DigitalOcean / Google Cloud / Render)

or Free options (Railway.app, Heroku free tier, PythonAnywhere)


This ensures it runs 24/7 without your laptop needing to stay on.



3. Polling for New Questions

The bot periodically checks Chegg’s account page (using cookie-based authentication).

If new questions are available → extract question details (title, link, subject, etc.).

Store last seen questions in a database or JSON file (so you don’t send duplicates).



4. Notification to Telegram

Use a Telegram Bot (via BotFather) to send you notifications in real-time.

Example: “📢 New Question Posted! [Link] Subject: Mechanical Engineering”.



5. Update Cookie Feature

You send a command in your Telegram bot: /updatecookie <cookie_string>

The backend updates the stored cookie file with your new one.

Next requests use the updated cookie (no need to redeploy code).





---

🔹 Tech Stack Recommendation

Language: Python (easiest for scraping + Telegram bot)

Libraries:

requests (to fetch Chegg with cookie)

bs4 (BeautifulSoup for scraping HTML)

python-telegram-bot (send notifications to Telegram)

schedule or APScheduler (to check every X minutes)




---

🔹 Example Bot Flow (Python)

import requests
from bs4 import BeautifulSoup
from telegram import Bot
import time

# 🔹 Telegram Bot Setup
TELEGRAM_TOKEN = "YOUR_BOT_TOKEN"
CHAT_ID = "YOUR_CHAT_ID"
bot = Bot(token=TELEGRAM_TOKEN)

# 🔹 Cookie Storage (Update manually or via /updatecookie command later)
cookies = {
    "session": "YOUR_CHEGG_COOKIE_VALUE"
}

# 🔹 Function to Check Questions
def check_questions():
    url = "https://www.chegg.com/homework-help/questions-and-answers"
    response = requests.get(url, cookies=cookies)
    
    if response.status_code == 200:
        soup = BeautifulSoup(response.text, "html.parser")
        # Extract question links/titles (selector depends on Chegg’s HTML structure)
        questions = soup.find_all("a", href=True)
        for q in questions:
            link = q['href']
            title = q.text.strip()
            # Send Telegram notification
            bot.send_message(chat_id=CHAT_ID, text=f"📢 New Question:\n{title}\n🔗 {link}")
    else:
        bot.send_message(chat_id=CHAT_ID, text="❌ Cookie expired or invalid. Please update.")

# 🔹 Run Forever (check every 5 minutes)
while True:
    check_questions()
    time.sleep(300)  # 5 minutes


---

🔹 Deployment

Upload this script to:

Heroku / Railway.app (easy, free tier available)

VPS (DigitalOcean, AWS, etc.) for more control


It will run 24/7 and notify you instantly.



---

🔹 Cookie Update

Add a /updatecookie <new_cookie> command to your bot:

Store new cookie in a cookies.json file.

Reload before making Chegg requests.


That way, you just paste the fresh cookie into Telegram, and bot updates itself.



---

✅ This way:

No need to keep laptop open.

Works 24/7 in cloud.

Cookie can be updated anytime.

You’ll get instant Telegram notifications for new Chegg questions.



---

Do you want me to write the full Python project structure (with /updatecookie command + database for last seen questions + deployment guide on Railway/Heroku) so you can directly run it?

Nice — I’ll walk you through exactly how to build a 24/7 Chegg question-notification bot end-to-end: architecture, code (complete working example), cookie update flow, deployment, security, and reliability tips. I’ll aim for practical, copy-pasteable instructions so you can get this running on a cloud server (no laptop needed).


---

1) High level idea (one-sentence)

Run a small backend service in the cloud which (a) uses your Chegg session cookie to access your Chegg account pages, (b) looks for new questions periodically, and (c) sends instant notifications to your Telegram bot. You can update the cookie at any time from Telegram.


---

2) Components & responsibilities

Telegram bot — receives /updatecookie and sends notifications.

Scraper/poller — loads Chegg pages using your session cookie, extracts new question data.

Storage — simple DB (SQLite) to store last-seen question IDs + encrypted cookie.

Controller / background job — runs the poller every N minutes.

Deployment — runs on a VPS / cloud (DigitalOcean, Render, Railway, etc.) or Docker container with Playwright browsers installed.



---

3) Why use Playwright (recommended) vs requests

Chegg pages are dynamic (JS). requests often won’t show the actual question list.

Playwright (headless Chromium) loads pages like a real browser and accepts cookies copied from your browser. It’s more robust.

Drawback: larger container and dependencies — but totally OK on any small VPS or container.



---

4) Prereqs (locally and on server)

Python 3.10+

pip and ability to install system packages if using Playwright

Telegram account + Bot token (create via BotFather)

A server or cloud instance (DigitalOcean droplet, Render, Railway, etc.)

Optional: Docker for consistent environment



---

5) How to extract the cookie (brief & safe)

1. Open Chrome (or your browser) -> log in to Chegg.


2. DevTools -> Application tab -> Cookies -> select chegg.com.


3. Copy the cookies you see (usually a list of name=value pairs). Best is to copy the cookie string from a request header (Network tab -> click a request to Chegg -> Headers -> Request Headers -> cookie:).


4. Paste the cookie string into the bot using /updatecookie or upload a .txt file with that content.



> Security note: Only use cookies from accounts you own. Keep the cookie private.




---

6) Full working example (Python) — copy/pasteable

Install dependencies

python -m pip install playwright python-telegram-bot beautifulsoup4 cryptography
# install browsers and dependencies (once)
playwright install --with-deps

main.py
This simple script:

runs a Telegram bot,

accepts /updatecookie <cookie-string> or a .txt upload,

periodically (every POLL_INTERVAL seconds) uses Playwright to load Chegg and finds new questions,

saves seen question IDs to SQLite to avoid duplicates,

encrypts cookie in DB with a Fernet key from env.


# main.py
import os, time, threading, sqlite3, json
from datetime import datetime
from cryptography.fernet import Fernet
from bs4 import BeautifulSoup
from playwright.sync_api import sync_playwright
from telegram import Bot, Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes

# ========= CONFIG =========
TELEGRAM_TOKEN = os.environ.get("TELEGRAM_TOKEN")
ADMIN_TELEGRAM_ID = int(os.environ.get("ADMIN_TELEGRAM_ID", "0"))  # your telegram user id as integer
CHAT_ID = os.environ.get("CHAT_ID", str(ADMIN_TELEGRAM_ID))       # where to send notifications
POLL_INTERVAL = int(os.environ.get("POLL_INTERVAL", "180"))       # seconds, change as needed
FERNET_KEY = os.environ.get("FERNET_KEY")                        # must be set (see README below)
# ==========================

if not TELEGRAM_TOKEN or not FERNET_KEY or ADMIN_TELEGRAM_ID == 0:
    print("Please set TELEGRAM_TOKEN, FERNET_KEY and ADMIN_TELEGRAM_ID environment variables.")
    exit(1)

fernet = Fernet(FERNET_KEY.encode())

# --------- DB init ----------
conn = sqlite3.connect("bot.db", check_same_thread=False)
cur = conn.cursor()
cur.execute("""
CREATE TABLE IF NOT EXISTS cookies (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  token TEXT,
  updated_at TEXT
)
""")
cur.execute("""
CREATE TABLE IF NOT EXISTS questions (
  qid TEXT PRIMARY KEY,
  title TEXT,
  url TEXT,
  seen_at TEXT
)
""")
conn.commit()

# ---------- cookie storage ----------
def save_cookie_raw(raw_cookie_str: str):
    token = fernet.encrypt(raw_cookie_str.encode()).decode()
    cur.execute("DELETE FROM cookies")  # single cookie only
    cur.execute("INSERT INTO cookies (token, updated_at) VALUES (?,datetime('now'))", (token,))
    conn.commit()

def get_cookie_raw():
    cur.execute("SELECT token FROM cookies ORDER BY id DESC LIMIT 1")
    r = cur.fetchone()
    if not r:
        return None
    try:
        return fernet.decrypt(r[0].encode()).decode()
    except Exception as e:
        print("Decrypt error:", e)
        return None

# ---------- Playwright check function ----------
def parse_cookie_string_to_list(cookie_str):
    pairs = []
    for part in cookie_str.split(';'):
        p = part.strip()
        if not p or '=' not in p:
            continue
        name, value = p.split('=', 1)
        pairs.append({
            "name": name.strip(),
            "value": value.strip(),
            "domain": ".chegg.com",
            "path": "/"
        })
    return pairs

def check_chegg_for_new_questions():
    cookie_str = get_cookie_raw()
    if not cookie_str:
        print("[poller] no cookie stored yet.")
        return []

    new_items = []
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True, args=["--no-sandbox"])
        context = browser.new_context()
        cookies = parse_cookie_string_to_list(cookie_str)
        if cookies:
            context.add_cookies(cookies)
        page = context.new_page()
        # go to Chegg homework questions page
        target_url = "https://www.chegg.com/homework-help/questions-and-answers"
        try:
            page.goto(target_url, timeout=60000)
            page.wait_for_load_state("networkidle", timeout=60000)
        except Exception as e:
            print("[poller] page load error:", e)
            browser.close()
            return []

        html = page.content()
        soup = BeautifulSoup(html, "html.parser")

        # --- SELECTOR: broad anchor selector for Q links (you may need to tune this) ---
        anchors = soup.select('a[href*="/homework-help/questions-and-answers/"]')
        for a in anchors:
            href = a.get('href') or ""
            if href.startswith('/'):
                href = "https://www.chegg.com" + href
            title = a.get_text(strip=True) or "chegg question"
            # derive a qid from URL
            qid = href.split('/')[-1].split('?')[0]
            if not qid:
                continue
            # check DB: skip if seen
            cur.execute("SELECT 1 FROM questions WHERE qid=?", (qid,))
            if cur.fetchone():
                continue
            # new question -> store and return
            cur.execute("INSERT OR IGNORE INTO questions (qid,title,url,seen_at) VALUES (?,?,?,datetime('now'))",
                        (qid, title, href))
            conn.commit()
            new_items.append({"qid": qid, "title": title, "url": href})
        browser.close()
    return new_items

# ---------- Poller loop ----------
def poller_loop(bot: Bot):
    print("[poller] started, interval:", POLL_INTERVAL)
    while True:
        try:
            new = check_chegg_for_new_questions()
            for item in new:
                text = f"📢 *New Chegg Question*\n{item['title']}\n{item['url']}"
                try:
                    bot.send_message(chat_id=CHAT_ID, text=text, parse_mode='Markdown')
                except Exception as e:
                    print("Telegram send error:", e)
        except Exception as e:
            print("Poller error:", e)
        time.sleep(POLL_INTERVAL)

# ---------- Telegram handlers ----------
async def start_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Chegg-notify bot running. Use /updatecookie <cookie-string> or upload cookie .txt (admin only).")

async def updatecookie_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user.id
    if user != ADMIN_TELEGRAM_ID:
        await update.message.reply_text("Unauthorized.")
        return
    args = context.args
    if args:
        cookie = " ".join(args)
        save_cookie_raw(cookie)
        await update.message.reply_text("✅ Cookie updated (from command).")
    else:
        await update.message.reply_text("Send cookie text or upload a .txt file with cookie content.")

async def message_text_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # If owner sends plain text that looks like cookie -> save it
    user = update.effective_user.id
    if user != ADMIN_TELEGRAM_ID:
        return
    text = update.message.text or ""
    if '=' in text and 'chegg' in text.lower() or '=' in text and ';' in text:
        save_cookie_raw(text)
        await update.message.reply_text("✅ Cookie updated (from message).")

async def doc_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user.id
    if user != ADMIN_TELEGRAM_ID:
        return
    doc = update.message.document
    if not doc:
        return
    f = await doc.get_file()
    tmp = f"/tmp/{doc.file_unique_id}.txt"
    await f.download_to_drive(tmp)
    with open(tmp, "r", encoding="utf-8", errors="ignore") as fh:
        content = fh.read()
    save_cookie_raw(content)
    await update.message.reply_text("✅ Cookie updated (from uploaded file).")

# ---------- main ----------
def main():
    # Telegram setup
    app = ApplicationBuilder().token(TELEGRAM_TOKEN).build()
    app.add_handler(CommandHandler("start", start_cmd))
    app.add_handler(CommandHandler("updatecookie", updatecookie_cmd))
    app.add_handler(MessageHandler(filters.Document.ALL, doc_handler))
    app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), message_text_handler))

    bot = Bot(token=TELEGRAM_TOKEN)
    # start poller thread
    t = threading.Thread(target=poller_loop, args=(bot,), daemon=True)
    t.start()

    # start telegram polling (blocking)
    print("Starting Telegram polling...")
    app.run_polling()

if __name__ == "__main__":
    main()

Environment variables you MUST set before running

export TELEGRAM_TOKEN="12345:ABC..."      # from BotFather
export ADMIN_TELEGRAM_ID="111111111"     # your telegram id (integer)
export CHAT_ID="111111111"               # where notifications are sent (can be same)
export POLL_INTERVAL="180"               # seconds between checks
# generate a Fernet key once:
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
export FERNET_KEY="...put output here..."

Run

python main.py


---

7) Deployment recommendations (24/7)

Docker: build a container that installs Python packages and runs playwright install --with-deps at image build time. Use an official Playwright image or run the playwright installer in Dockerfile. Example base images exist (Playwright official).

VPS: run under systemd or pm2/supervisord and ensure restart on crash.

Managed hosts: Railway / Render / DigitalOcean Apps — ensure the container includes Playwright browsers and necessary system libs. If using a "serverless" platform, make sure it supports long-running processes.


Simple Dockerfile sketch:

FROM python:3.11-slim
RUN apt-get update && apt-get install -y wget gnupg ca-certificates \
    libgtk-3-0 libx11-6 libx11-xcb1 libasound2 libnss3 libxss1 libgbm1 \
    libatk1.0-0 libpangocairo-1.0-0 libcups2 libxcomposite1 libxrandr2 \
    libxdamage1 libxinerama1 libxkbcommon0
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . /app
RUN playwright install --with-deps
CMD ["python", "main.py"]


---

8) Security & reliability (important)

Encrypt cookie: we used Fernet — store FERNET_KEY securely on server env.

Lock /updatecookie: only the admin Telegram id can change cookie.

Never print cookie to logs.

Rate limits: set POLL_INTERVAL realistically (e.g., 2–5 minutes). Too frequent requests may flag the account.

User agent & human-like browsing (optional): when using Playwright, set a realistic viewport and user agent and small delays to avoid detection.

Backup DB: copy bot.db periodically or push to remote storage.

Monitoring: use a simple health check endpoint or a log that indicates last successful run; restart automatically (docker restart policies or systemd).



---

9) Handling cookie expiry & reauth problems

If Chegg invalidates the cookie the poller will either get redirected to login or not show question list. Detect this (e.g., check for a login indicator) and notify you by Telegram: “Cookie expired — please update”.

Then update cookie via /updatecookie with a fresh cookie from your browser.



---

10) Selector tuning & robustness

The example uses a broad anchor selector: a[href*="/homework-help/questions-and-answers/"]. Chegg may change HTML; you’ll need to inspect the page and update the CSS selector to the actual question-card anchors or use page DOM attributes that identify question cards.

To debug selectors: print page.content() locally and test BeautifulSoup selectors.



---

11) Extra features you can add later

Store question full HTML and do fuzzy matching to avoid duplicates if URLs change.

Provide /last command to list last N notifications.

Provide channels/groups support (send to multiple chat IDs).

Web UI to view stored questions and change settings.

Use webhooks instead of polling (requires Chegg to offer a webhook — unlikely).



---

12) Legal / TOS & ethics

Automating access to a website may violate the website’s Terms of Service. Use this system only on accounts you control and accept the risk of temporary blocks or account action. If you’re unsure, check Chegg’s terms.



---

13) Final checklist to get started right now

1. Create Telegram bot and note token.


2. Get your Telegram user ID (search @userinfobot on Telegram).


3. Generate a FERNET_KEY and keep it safe.


4. Create a small server (DigitalOcean droplet, Render, Railway).


5. Copy the main.py above and set environment variables on the server.


6. Install dependencies and Playwright browsers on the server.


7. Run python main.py.


8. Update cookie via /updatecookie (paste cookie or upload .txt) from your Telegram account.


9. Wait for notifications when new questions appear.




---

If you want, I can do any of these next steps for you right now:

adapt the selector to the exact Chegg page HTML you see (paste a sample HTML snippet or a screenshot of the page DOM),

create a Dockerfile + deployment guide for Render/Railway tailored to Playwright, or

provide a cleaner Web UI (FastAPI + simple HTML) for cookie updates and viewing recent notifications.


Which one do you want me to deliver next?

