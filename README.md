import telebot
from telebot import types
from datetime import datetime, timedelta
import json
import os
import time

# ===============================
# الإعدادات
# ===============================
MAIN_BOT_TOKEN = "8171819063:AAHR7QwHWVBo4MPta1jNjNYaMBC06qDdnq0"  # بوت الخدمات
CONTROL_BOT_TOKEN = "8121717175:AAETKRzNzctKx2pfSqFp-2WT1DNUqXkOOaM"  # بوت التحكم الثاني
ADMIN_ID = 7144135936
DEVELOPER = "xxcc_2@"  # اسم المطور

DATA_FILE = "data.json"

# بيانات الخدمات
config = {
    "prices": {"followers": 38, "likes": 10, "views": 5},
    "limits": {"followers": 100, "likes": 500, "views": 50000},
    "services_status": {"followers": True, "likes": True, "views": True}
}

DAILY_GIFT = 100

# ===============================
# بيانات المستخدمين
# ===============================
def load_data():
    if os.path.exists(DATA_FILE):
        return json.load(open(DATA_FILE, "r"))
    return {}

def save_data():
    json.dump(data, open(DATA_FILE, "w"), indent=2)

data = load_data()

def user(uid):
    uid = str(uid)
    if uid not in data:
        data[uid] = {"points": 0, "last_daily": None, "invites": 0}
        save_data()
    return data[uid]

def can_daily(uid):
    info = user(uid)
    last = info["last_daily"]
    if not last:
        return True
    try:
        return datetime.now() - datetime.fromisoformat(last) >= timedelta(hours=24)
    except:
        info["last_daily"] = None
        return True

# ===============================
# إنشاء البوتات
# ===============================
main_bot = telebot.TeleBot(MAIN_BOT_TOKEN, parse_mode="HTML")
control_bot = telebot.TeleBot(CONTROL_BOT_TOKEN, parse_mode="HTML")

# ===============================
# الواجهات
# ===============================
def main_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add(
        types.KeyboardButton("💎 الخدمات"),
        types.KeyboardButton("🎁 هديتي اليومية"),
        types.KeyboardButton("📌 نقاطي"),
        types.KeyboardButton("🔗 رابط الدعوة"),
        types.KeyboardButton("👤 حسابي"),
        types.KeyboardButton("👨‍💻 المطور")
    )
    return markup

def services_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    for s in config["prices"]:
        if config["services_status"].get(s, True):
            emoji = {"followers":"👥", "likes":"❤️", "views":"👀"}[s]
            markup.add(types.KeyboardButton(f"{emoji} {s.capitalize()}"))
    markup.add(types.KeyboardButton("⬅️ رجوع"))
    return markup

def admin_panel():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    markup.add(
        types.KeyboardButton("💰 تعديل الأسعار"),
        types.KeyboardButton("🔄 تفعيل/إيقاف الخدمات"),
        types.KeyboardButton("⬅️ رجوع")
    )
    return markup

# ===============================
# وظائف لوحة التحكم
# ===============================
def handle_control_admin(msg):
    text = msg.text.lower()
    if text == "💰 تعديل الأسعار":
        control_bot.send_message(msg.chat.id, "✏️ أرسل الخدمة والسعر الجديد بالصيغة: followers 50")
    elif text == "🔄 تفعيل/إيقاف الخدمات":
        control_bot.send_message(msg.chat.id, "⚙️ أرسل الخدمة لتفعيلها أو إيقافها: likes true/false")
    elif " " in text:
        parts = text.split()
        service = parts[0]
        if service in config["prices"]:
            if parts[1].isdigit():
                config["prices"][service] = int(parts[1])
                control_bot.send_message(msg.chat.id, f"✅ تم تحديث سعر {service}")
            elif parts[1].lower() in ["true","false"]:
                config["services_status"][service] = parts[1].lower() == "true"
                control_bot.send_message(msg.chat.id, f"✅ تم تعديل حالة {service}")
    elif text == "⬅️ رجوع":
        control_bot.send_message(msg.chat.id, "↩️ تم الرجوع للقائمة الرئيسية")

# ===============================
# بوت التحكم
# ===============================
@control_bot.message_handler(func=lambda m: m.from_user.id == ADMIN_ID)
def control_commands(msg):
    handle_control_admin(msg)

# ===============================
# بوت المستخدمين
# ===============================
@main_bot.message_handler(commands=['start'])
def start(msg):
    user(msg.from_user.id)
    main_bot.send_message(msg.chat.id, f"👋 أهلاً بك\n🎛 بوت خدمات احترافي", reply_markup=main_menu())

@main_bot.message_handler(func=lambda m: True)
def user_messages(msg):
    try:
        uid = msg.from_user.id
        info = user(uid)
        text = msg.text

        if text == "💎 الخدمات":
            main_bot.send_message(msg.chat.id, "📌 اختر الخدمة:", reply_markup=services_menu())
        elif text.lower() in ["👥 followers", "❤️ likes", "👀 views"]:
            service = {"👥 followers":"followers", "❤️ likes":"likes", "👀 views":"views"}[text.lower()]
            info["service"] = service
            main_bot.send_message(msg.chat.id, f"✔️ اختر الرابط الآن للخدمة: {text}")
        elif "http" in text:
            if "service" not in info:
                main_bot.send_message(msg.chat.id, "❌ اختر الخدمة أولاً")
                return
            info["link"] = text
            main_bot.send_message(msg.chat.id, "✔️ تم استلام الرابط، أرسل الكمية الآن:")
        elif text.isdigit():
            if "service" not in info or "link" not in info:
                main_bot.send_message(msg.chat.id, "❌ اختر الخدمة وارسل الرابط أولاً")
                return
            service = info["service"]
            amount = int(text)
            if amount > config["limits"][service]:
                main_bot.send_message(msg.chat.id, f"❌ الحد الأقصى: {config['limits'][service]}")
                return
            price = config["prices"][service] * amount
            if info["points"] < price:
                main_bot.send_message(msg.chat.id, f"❌ نقاطك لا تكفي\nالسعر المطلوب: {price}")
                return
            info["points"] -= price
            main_bot.send_message(msg.chat.id, f"🚀 تم تقديم الطلب:\nالخدمة: {service}\nالكمية: {amount}\nالسعر: {price}\nالرابط: {info['link']}")
        elif text == "🎁 هديتي اليومية":
            if not can_daily(uid):
                main_bot.send_message(msg.chat.id, "❌ أخذت الهدية اليوم")
                return
            info["points"] += DAILY_GIFT
            info["last_daily"] = datetime.now().isoformat()
            main_bot.send_message(msg.chat.id, f"🎁 تم منحك {DAILY_GIFT} نقطة!")
        elif text == "📌 نقاطي":
            main_bot.send_message(msg.chat.id, f"💰 نقاطك: {info['points']}")
        elif text == "🔗 رابط الدعوة":
            main_bot.send_message(msg.chat.id, f"🔗 رابط دعوتك:\nt.me/{main_bot.get_me().username}?start={uid}")
        elif text == "👤 حسابي":
            main_bot.send_message(msg.chat.id, f"👤 بياناتك:\nالنقاط: {info['points']}\nالدعوات: {info['invites']}")
        elif text == "👨‍💻 المطور":
            main_bot.send_message(msg.chat.id, f"👨‍💻 المطور: {DEVELOPER}")

        save_data()
    except Exception as e:
        print(f"❌ حدث خطأ: {e}")
        main_bot.send_message(msg.chat.id, "❌ حدث خطأ داخلي، يرجى المحاولة مجددًا")

# ===============================
# تشغيل البوتين دائمًا
# ===============================
while True:
    try:
        main_bot.infinity_polling()
    except Exception as e:
        print(f"❌ حدث خطأ في بوت الخدمات: {e}\n⏳ إعادة التشغيل خلال 5 ثواني...")
        time.sleep(5)

    try:
        control_bot.infinity_polling()
    except Exception as e:
        print(f"❌ حدث خطأ في بوت التحكم: {e}\n⏳ إعادة التشغيل خلال 5 ثواني...")
        time.sleep(5)
        
