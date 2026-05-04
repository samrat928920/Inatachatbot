from telegram import Update
from telegram.ext import ApplicationBuilder, MessageHandler, CommandHandler, filters, ContextTypes
import requests
import time
import os

# 🔐 SAFE KEYS (Railway env se aayega)
TOKEN = os.getenv("TOKEN")
GEMINI_API_KEY = os.getenv("GEMINI_API_KEY")

# 🧠 memory
user_memory = {}

# ⏱️ anti-spam
last_used = {}

def can_reply(user_id):
    now = time.time()
    if user_id in last_used and now - last_used[user_id] < 2:
        return False
    last_used[user_id] = now
    return True

# 🤖 Gemini AI
def get_reply(prompt):
    try:
        url = f"https://generativelanguage.googleapis.com/v1/models/gemini-2.5-flash:generateContent?key={GEMINI_API_KEY}"

        data = {
            "contents": [
                {
                    "parts": [{"text": prompt}]
                }
            ]
        }

        res = requests.post(url, json=data, timeout=15)
        result = res.json()

        print("DEBUG:", result)

        if "candidates" in result:
            return result["candidates"][0]["content"]["parts"][0]["text"]

        if "error" in result:
            return "API Error: " + result["error"]["message"]

        return "Samajh nahi aaya 😅"

    except Exception as e:
        print("Error:", e)
        return "Network issue 😢"

# 📌 /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Hello bhai 😎 Main Gemini 2.5 Flash AI bot hoon!")

# 📌 /help
async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Bas message bhejo 😄 main Hinglish me reply dunga 🤖")

# 💬 chat handler
async def reply(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message and update.message.text:

        user_id = update.message.chat_id
        user_msg = update.message.text

        if not can_reply(user_id):
            return

        history = user_memory.get(user_id, "")

        prompt = (
            "You are a friendly Indian chatbot. "
            "Always reply in Hinglish (Hindi written in English letters). "
            "Keep replies short and natural.\n\n"
            f"{history}\nUser: {user_msg}"
        )

        bot_reply = get_reply(prompt)

        user_memory[user_id] = (history + f"\nUser: {user_msg}\nBot: {bot_reply}")[-3000:]

        print(user_msg, "->", bot_reply)

        await update.message.reply_text(bot_reply)

# 🚀 APP SETUP
app = ApplicationBuilder().token(TOKEN).build()

app.add_handler(CommandHandler("start", start))
app.add_handler(CommandHandler("help", help_cmd))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, reply))

print("🤖 Bot RUNNING (Secure Mode)")
app.run_polling() 