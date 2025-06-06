# Refer-and-earn 

import sqlite3
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder, CommandHandler, CallbackQueryHandler, MessageHandler,
    ContextTypes, filters
)

# === DB SETUP ===
conn = sqlite3.connect("bot_data.db", check_same_thread=False)
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    balance INTEGER DEFAULT 0,
    verified INTEGER DEFAULT 0,
    referrer INTEGER
)
""")
conn.commit()

# === HELPER FUNCTIONS ===
def add_user(user_id, referrer=None):
    cursor.execute("INSERT OR IGNORE INTO users (user_id, referrer) VALUES (?, ?)", (user_id, referrer))
    conn.commit()

def is_verified(user_id):
    cursor.execute("SELECT verified FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    return row and row[0] == 1

def mark_verified(user_id):
    cursor.execute("UPDATE users SET verified = 1 WHERE user_id = ?", (user_id,))
    conn.commit()

def add_balance(user_id, amount):
    cursor.execute("UPDATE users SET balance = balance + ? WHERE user_id = ?", (amount, user_id))
    conn.commit()

def get_balance(user_id):
    cursor.execute("SELECT balance FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    return row[0] if row else 0

# === TELEGRAM BOT SETUP ===
CHANNELS = [
    "@loottheamazon",
    "@safesecurecarding",
    "@BGMIHACKWALA1",
    "@jalwa_game_official6"
]

# /start command
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    args = context.args

    # Track referral
    referrer = None
    if args and args[0].startswith("ref_"):
        try:
            referrer = int(args[0].replace("ref_", ""))
        except:
            pass

    add_user(user_id, referrer)

    if referrer and referrer != user_id:
        add_balance(referrer, 10)

    if is_verified(user_id):
        await show_main_menu(update, context)
    else:
        await ask_to_join_channels(update)

# Show join buttons
async def ask_to_join_channels(update: Update):
    keyboard = [[InlineKeyboardButton("Join " + ch, url=f"https://t.me/{ch[1:]}")] for ch in CHANNELS]
    keyboard.append([InlineKeyboardButton("✅ I've Joined All", callback_data="check_join")])
    reply_markup = InlineKeyboardMarkup(keyboard)

    await update.message.reply_text("📢 Please join all channels to continue:", reply_markup=reply_markup)

# Main menu
async def show_main_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("Refer and Earn", callback_data="refer")],
        [InlineKeyboardButton("Balance", callback_data="balance")],
        [InlineKeyboardButton("Withdraw", callback_data="withdraw")]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)

    if update.message:
        await update.message.reply_text("✅ You're verified!\nChoose an option:", reply_markup=reply_markup)
    elif update.callback_query:
        await update.callback_query.message.reply_text("✅ You're verified!\nChoose an option:", reply_markup=reply_markup)

# Callback button handler
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = query.from_user.id
    await query.answer()

    if query.data == "check_join":
        for channel in CHANNELS:
            member = await context.bot.get_chat_member(chat_id=channel, user_id=user_id)
            if member.status not in ["member", "administrator", "creator"]:
                await query.edit_message_text("❌ Please join all channels to continue.")
                return
        mark_verified(user_id)
        await query.edit_message_text("✅ All channels joined. Welcome!")
        await show_main_menu(update, context)

    elif query.data == "refer":
        link = f"https://t.me/{context.bot.username}?start=ref_{user_id}"
        await query.edit_message_text(f"🔗 Your referral link:\n{link}\nEarn ₹10 per referral.")

    elif query.data == "balance":
        balance = get_balance(user_id)
        await query.edit_message_text(f"💰 Your balance: ₹{balance}")

    elif query.data == "withdraw":
        balance = get_balance(user_id)
        if balance >= 110:
            context.user_data["awaiting_upi"] = True
            await query.edit_message_text("Please enter your UPI ID to receive ₹110:")
        else:
            await query.edit_message_text("❌ You need at least ₹110 to withdraw.")

# UPI handler
async def upi_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if context.user_data.get("awaiting_upi"):
        upi = update.message.text
        context.user_data["awaiting_upi"] = False
        cursor.execute("UPDATE users SET balance = balance - 110 WHERE user_id = ?", (user_id,))
        conn.commit()
        await update.message.reply_text(f"✅ Withdrawal requested. ₹110 will be sent to: {upi}")
    else:
        await update.message.reply_text("Please use the menu to navigate.")

# === APP BUILD ===
app = ApplicationBuilder().token("7713010651:AAGjKJ8wxOBO-lKs8DmaUDrhaWADL5SpJUI").build()

app.add_handler(CommandHandler("start", start))
app.add_handler(CallbackQueryHandler(button_handler))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, upi_handler))

app.run_polling()
