import telebot
from telebot import types
import json
import os
import datetime

# ================= CONFIG =================
BOT_TOKEN = "8534427928:AAEeOKXj4L8bpkvw2FiOcwuQiPdwR5c9aOg"
ADMIN_ID = 5828992083

bot = telebot.TeleBot(BOT_TOKEN)

# ================= FILES =================
USERS_FILE = "users.json"
TASKS_FILE = "tasks.json"
SETTINGS_FILE = "settings.json"
WITHDRAW_FILE = "withdraw.json"

# ================= JSON =================
def load_json(file, default):
    if not os.path.exists(file):
        with open(file, "w") as f:
            json.dump(default, f)
        return default
    with open(file, "r") as f:
        return json.load(f)

def save_json(file, data):
    with open(file, "w") as f:
        json.dump(data, f, indent=4)

def get_users(): return load_json(USERS_FILE, {})
def save_users(data): save_json(USERS_FILE, data)

def get_tasks(): return load_json(TASKS_FILE, [])
def save_tasks(data): save_json(TASKS_FILE, data)

def get_settings():
    return load_json(SETTINGS_FILE, {
        "refer_bonus": 5,
        "daily_bonus": 1,
        "min_withdraw": 50,
        "currency": "৳",
        "channels": []
    })

def get_withdraws(): return load_json(WITHDRAW_FILE, [])
def save_withdraws(data): save_json(WITHDRAW_FILE, data)

# ================= STATES =================
user_states = {}
temp_data = {}

# ================= JOIN CHECK =================
def check_join(uid):
    settings = get_settings()
    for ch in settings["channels"]:
        try:
            status = bot.get_chat_member(ch, uid).status
            if status not in ["member", "administrator", "creator"]:
                return False
        except:
            return False
    return True

# ================= MENU =================
def menu(uid):
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("💰 Balance", "🎁 Daily Bonus")
    kb.add("📋 Tasks", "👫 Refer")
    kb.add("💳 Withdraw", "🏆 Leaderboard")

    if uid == ADMIN_ID:
        kb.add("🔐 Admin Panel")

    return kb

# ================= START =================
@bot.message_handler(commands=["start"])
def start(msg):
    uid = str(msg.chat.id)
    users = get_users()
    settings = get_settings()
    args = msg.text.split()

    if uid not in users:
        users[uid] = {
            "name": msg.from_user.first_name,
            "balance": 0,
            "refer": 0,
            "completed": [],
            "last_bonus": None
        }

        # referral
        if len(args) > 1:
            ref = args[1]
            if ref in users and ref != uid:
                users[ref]["balance"] += settings["refer_bonus"]
                users[ref]["refer"] += 1

    save_users(users)

    if not check_join(int(uid)):
        bot.send_message(uid, "⚠️ Join required channels")
        return

    bot.send_message(uid, "👋 Welcome!", reply_markup=menu(int(uid)))

# ================= BALANCE =================
@bot.message_handler(func=lambda m: m.text == "💰 Balance")
def balance(msg):
    users = get_users()
    settings = get_settings()
    uid = str(msg.chat.id)

    bot.send_message(uid, f"💰 {users[uid]['balance']} {settings['currency']}")

# ================= DAILY BONUS =================
@bot.message_handler(func=lambda m: m.text == "🎁 Daily Bonus")
def bonus(msg):
    users = get_users()
    settings = get_settings()
    uid = str(msg.chat.id)

    today = str(datetime.date.today())

    if users[uid]["last_bonus"] == today:
        bot.send_message(uid, "❌ Already claimed")
        return

    users[uid]["balance"] += settings["daily_bonus"]
    users[uid]["last_bonus"] = today
    save_users(users)

    bot.send_message(uid, "🎁 Bonus added")

# ================= REFER =================
@bot.message_handler(func=lambda m: m.text == "👫 Refer")
def refer(msg):
    users = get_users()
    uid = str(msg.chat.id)

    link = f"https://t.me/{bot.get_me().username}?start={uid}"
    bot.send_message(uid, f"🔗 Link:\n{link}\n👥 Refers: {users[uid]['refer']}")

# ================= TASKS =================
@bot.message_handler(func=lambda m: m.text == "📋 Tasks")
def tasks(msg):
    tasks = get_tasks()
    users = get_users()
    uid = str(msg.chat.id)

    if not tasks:
        bot.send_message(uid, "❌ No tasks")
        return

    kb = types.InlineKeyboardMarkup()

    for t in tasks:
        if t["id"] not in users[uid]["completed"]:
            kb.add(types.InlineKeyboardButton(
                f"{t['title']} 💰{t['reward']}",
                callback_data=f"task_{t['id']}"
            ))

    bot.send_message(uid, "📋 Tasks:", reply_markup=kb)

# ================= WITHDRAW =================
@bot.message_handler(func=lambda m: m.text == "💳 Withdraw")
def withdraw(msg):
    users = get_users()
    settings = get_settings()
    uid = str(msg.chat.id)

    if users[uid]["balance"] < settings["min_withdraw"]:
        bot.send_message(uid, "❌ Minimum not reached")
        return

    bot.send_message(uid, "📱 Send number:")
    user_states[uid] = "number"

# ================= STATE =================
@bot.message_handler(func=lambda m: True)
def state(msg):
    uid = str(msg.chat.id)

    if user_states.get(uid) == "number":
        temp_data[uid] = {"number": msg.text}
        bot.send_message(uid, "💰 Enter amount:")
        user_states[uid] = "amount"

    elif user_states.get(uid) == "amount":
        try:
            amount = float(msg.text)
        except:
            bot.send_message(uid, "❌ Invalid")
            return

        users = get_users()

        if amount <= 0 or amount > users[uid]["balance"]:
            bot.send_message(uid, "❌ Invalid")
            return

        users[uid]["balance"] -= amount
        save_users(users)

        wid = int(datetime.datetime.now().timestamp())

        withdraws = get_withdraws()
        withdraws.append({
            "id": wid,
            "user": uid,
            "amount": amount,
            "number": temp_data[uid]["number"],
            "status": "pending"
        })
        save_withdraws(withdraws)

        kb = types.InlineKeyboardMarkup()
        kb.add(
            types.InlineKeyboardButton("✅ Approve", callback_data=f"approve_{wid}"),
            types.InlineKeyboardButton("❌ Reject", callback_data=f"reject_{wid}")
        )

        bot.send_message(ADMIN_ID,
            f"💳 Withdraw\nUser: {uid}\nAmount: {amount}\nNumber: {temp_data[uid]['number']}",
            reply_markup=kb
        )

        bot.send_message(uid, "✅ Sent")
        user_states[uid] = None

# ================= CALLBACK =================
@bot.callback_query_handler(func=lambda call: True)
def callback(call):
    users = get_users()

    # TASK SYSTEM
    if call.data.startswith("task_"):
        tid = int(call.data.split("_")[1])
        tasks = get_tasks()
        uid = str(call.from_user.id)

        for t in tasks:
            if t["id"] == tid:

                if tid in users[uid]["completed"]:
                    bot.answer_callback_query(call.id, "Already done")
                    return

                users[uid]["balance"] += t["reward"]
                users[uid]["completed"].append(tid)
                save_users(users)

                bot.answer_callback_query(call.id, "Done")
                bot.send_message(uid, f"🎉 Earned {t['reward']}")
                break

    # ADMIN WITHDRAW
    if call.from_user.id != ADMIN_ID:
        return

    action, wid = call.data.split("_")
    wid = int(wid)

    withdraws = get_withdraws()

    for w in withdraws:
        if w["id"] == wid and w["status"] == "pending":

            if action == "approve":
                w["status"] = "approved"
                bot.send_message(w["user"], f"✅ Approved {w['amount']}")

            elif action == "reject":
                w["status"] = "rejected"
                users[w["user"]]["balance"] += w["amount"]
                bot.send_message(w["user"], "❌ Rejected & refunded")

            break

    save_withdraws(withdraws)
    save_users(users)
    bot.answer_callback_query(call.id, "Done")

# ================= LEADERBOARD =================
@bot.message_handler(func=lambda m: m.text == "🏆 Leaderboard")
def leaderboard(msg):
    users = get_users()
    settings = get_settings()

    top = sorted(users.items(), key=lambda x: x[1]["balance"], reverse=True)[:10]

    text = "🏆 Top Users\n\n"
    for i, u in enumerate(top):
        text += f"{i+1}. {u[1]['name']} - {u[1]['balance']} {settings['currency']}\n"

    bot.send_message(msg.chat.id, text)

# ================= RUN =================
print("Bot Running...")
bot.infinity_polling()
