import os
import uuid
import requests
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ContextTypes,
    filters,
)

# توکن ربات تلگرام خود را اینجا قرار دهید
TOKEN = "8881876296:AAHXGB0zA8o-7Mv2jP6vQngotGBTLURlmVA"

# حافظه‌ها
user_files = {}         
stored_contents = {}    
user_pending = {}       
user_password_wait = {} 
user_mode = {}          

def get_main_keyboard():
    keyboard = [
        [
            InlineKeyboardButton("🔗 آپلود در سرور خارجی", callback_data="mode_upload"),
            InlineKeyboardButton("📝 ساخت لینک داخلی (تلگرام)", callback_data="mode_telegram_link")
        ],
        [
            InlineKeyboardButton("👨‍💻 پشتیبانی", url="https://t.me/pe_she"),
            InlineKeyboardButton("ℹ️ راهنما", callback_data="btn_help")
        ]
    ]
    return InlineKeyboardMarkup(keyboard)

def get_server_choice_keyboard():
    keyboard = [
        [
            InlineKeyboardButton("🌐 آپلود در GoFile", callback_data="upload_gofile"),
            InlineKeyboardButton("⏱ آپلود در TmpFiles", callback_data="upload_tmpfiles")
        ],
        [InlineKeyboardButton("🔙 بازگشت به منوی اصلی", callback_data="btn_start")]
    ]
    return InlineKeyboardMarkup(keyboard)

def get_options_keyboard():
    keyboard = [
        [InlineKeyboardButton("🔓 لینک معمولی (بدون رمز)", callback_data="opt_normal")],
        [InlineKeyboardButton("🔐 لینک رمزدار (Password)", callback_data="opt_password")],
        [InlineKeyboardButton("💥 لینک یک‌‌بار مصرف (Burn After Read)", callback_data="opt_burn")],
        [InlineKeyboardButton("🔙 لغو و بازگشت", callback_data="btn_start")]
    ]
    return InlineKeyboardMarkup(keyboard)

def get_back_keyboard():
    keyboard = [
        [InlineKeyboardButton("🔙 بازگشت به منوی اصلی", callback_data="btn_start")]
    ]
    return InlineKeyboardMarkup(keyboard)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    
    # بررسی کلیک روی لینک‌های اختصاصی (Deep Link)
    if context.args:
        payload = context.args[0]
        if payload in stored_contents:
            data = stored_contents[payload]
            
            # اگر لینک رمز داشت
            if data.get("password"):
                user_password_wait[user_id] = {"payload": payload, "password": data["password"]}
                await update.message.reply_text("🔐 <b>این لینک محافظت شده است.</b>\n\nلطفاً رمز عبور را ارسال کنید:", parse_mode="HTML")
                return
            
            await send_stored_content_by_message(update.message, payload)
            return
        else:
            await update.message.reply_text("❌ متأسفانه این لینک معتبر نیست یا منقضی (سوخته) شده است.")
            return

    user_mode[user_id] = "main"
    welcome_text = (
        "<b>ربات لینک‌ساز پیشرفته فایل و متن 🔗</b>\n\n"
        "هر نوع لینک، عکس، فایل، ویدیو یا متنی را بفرستید تا لینک اختصاصی با قابلیت رمز و یک‌بار مصرف بسازیم!"
    )

    if update.message:
        await update.message.reply_text(welcome_text, parse_mode="HTML", reply_markup=get_main_keyboard())
    elif update.callback_query:
        try:
            await update.callback_query.message.edit_text(welcome_text, parse_mode="HTML", reply_markup=get_main_keyboard())
        except:
            await update.callback_query.message.reply_text(welcome_text, parse_mode="HTML", reply_markup=get_main_keyboard())

async def send_stored_content_by_message(message, payload: str):
    if payload not in stored_contents:
        await message.reply_text("❌ این لینک دیگر معتبر نیست یا منقضی شده است.")
        return
        
    data = stored_contents[payload]
    content_type = data["type"]
    
    await message.reply_text("📄 <b>محتوای لینک:</b>", parse_mode="HTML")
    
    if content_type == "text":
        await message.reply_text(data["content"])
    elif content_type == "photo":
        await message.reply_photo(data["content"], caption=data.get("caption", ""))
    elif content_type == "document":
        await message.reply_document(data["content"], caption=data.get("caption", ""))
    elif content_type == "video":
        await message.reply_video(data["content"], caption=data.get("caption", ""))
    elif content_type == "audio":
        await message.reply_audio(data["content"], caption=data.get("caption", ""))
    elif content_type == "voice":
        await message.reply_voice(data["content"])

    # اگر لینک یک‌بار مصرف بود، پاکش کن
    if data.get("burn", False):
        del stored_contents[payload]

# --- سرویس‌های آپلود خارجی ---
def upload_to_gofile(file_path: str) -> str:
    server_response = requests.get('https://api.gofile.io/servers')
    if server_response.status_code != 200:
        raise Exception("خطا در سرور GoFile")
    best_server = server_response.json()['data']['servers'][0]['name']
    with open(file_path, 'rb') as f:
        res = requests.post(f"https://{best_server}.gofile.io/contents/uploadfile", files={'file': f}).json()
    if res.get('status') == 'ok':
        return res['data']['downloadPage']
    raise Exception("خطا در آپلود GoFile")

def upload_to_tmpfiles(file_path: str) -> str:
    with open(file_path, 'rb') as f:
        res = requests.post('https://tmpfiles.org/api/v1/upload', files={'file': f})
    if res.status_code == 200:
        return res.json()['data']['url'].replace("tmpfiles.org/", "tmpfiles.org/dl/")
    raise Exception("خطا در آپلود TmpFiles")

# --- مدیریت دکمه‌ها ---
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id

    if query.data == "btn_start":
        user_pending.pop(user_id, None)
        user_password_wait.pop(user_id, None)
        await start(update, context)

    elif query.data == "mode_upload":
        user_mode[user_id] = "upload"
        await query.message.edit_text("📤 لطفاً فایل خود را جهت آپلود در سرور خارجی بفرستید:", reply_markup=get_back_keyboard())

    elif query.data == "mode_telegram_link":
        user_mode[user_id] = "telegram_link"
        await query.message.edit_text("📝 متن، عکس، یا فایل خود را بفرستید تا لینک اختصاصی تلگرام بسازیم:", reply_markup=get_back_keyboard())

    elif query.data == "btn_help":
        await query.message.edit_text("<b>راهنما:</b>\nمی‌توانید لینک‌های رمزدار یا یک‌بار مصرف بسازید.", parse_mode="HTML", reply_markup=get_main_keyboard())

    elif query.data in ["opt_normal", "opt_burn"]:
        if user_id not in user_pending:
            await query.message.edit_text("❌ خطایی رخ داد. لطفاً دوباره تلاش کنید.", reply_markup=get_main_keyboard())
            return
        
        unique_id = uuid.uuid4().hex[:8]
        content_data = user_pending[user_id]
        content_data["burn"] = (query.data == "opt_burn")
        content_data["password"] = None
        
        stored_contents[unique_id] = content_data
        user_pending.pop(user_id, None)

        bot_username = context.bot.username
        link = f"https://t.me/{bot_username}?start={unique_id}"
        burn_text = "💥 <b>نوع لینک:</b> یک‌بار مصرف (پس از اولین بازدید منقضی می‌شود)\n\n" if query.data == "opt_burn" else ""
        
        await query.message.edit_text(
            f"✅ <b>لینک شما آماده شد!</b>\n\n{burn_text}🔗 <b>لینک اختصاصی:</b>\n{link}",
            parse_mode="HTML",
            reply_markup=get_back_keyboard()
        )

    elif query.data == "opt_password":
        user_mode[user_id] = "waiting_for_password_creation"
        await query.message.edit_text(
            "🔐 لطفاً <b>رمز عبور</b> مورد نظر خود را (به صورت متن یا عدد) ارسال کنید:",
            parse_mode="HTML",
            reply_markup=get_back_keyboard()
        )

    elif query.data in ["upload_gofile", "upload_tmpfiles"]:
        file_path = user_files.get(user_id)
        if not file_path or not os.path.exists(file_path):
            await query.message.edit_text("❌ فایلی یافت نشد.")
            return
        server_name = "GoFile" if query.data == "upload_gofile" else "TmpFiles"
        await query.message.edit_text(f"⏳ در حال آپلود در {server_name}...")
        try:
            url = upload_to_gofile(file_path) if query.data == "upload_gofile" else upload_to_tmpfiles(file_path)
            await query.message.edit_text(f"✅ آپلود شد!\n\n🔗 لینک:\n{url}", reply_markup=get_main_keyboard())
        except Exception as e:
            await query.message.edit_text(f"⚠️ خطا: {e}")
        finally:
            if user_id in user_files and os.path.exists(user_files[user_id]):
                os.remove(user_files[user_id])
                user_files.pop(user_id, None)

# --- پردازش محتوا و رمزها ---
async def handle_incoming_content(update: Update, context: ContextTypes.DEFAULT_TYPE):
    message = update.message
    user_id = message.from_user.id
    text = message.text

    # ۱. اگر کاربر منتظر وارد کردن رمز برای باز کردن لینک است
    if user_id in user_password_wait:
        pass_data = user_password_wait[user_id]
        if text == pass_data["password"]:
            payload = pass_data["payload"]
            user_password_wait.pop(user_id, None)
            await send_stored_content_by_message(message, payload)
        else:
            await message.reply_text("❌ رمزت  غلطه گل درست بزن:")
        return

    # ۲. اگر کاربر در حال ساخت رمز برای لینک خودش است
    if user_mode.get(user_id) == "waiting_for_password_creation":
        password = text
        if user_id in user_pending:
            unique_id = uuid.uuid4().hex[:8]
            content_data = user_pending[user_id]
            content_data["password"] = password
            content_data["burn"] = False
            
            stored_contents[unique_id] = content_data
            user_pending.pop(user_id, None)
            user_mode[user_id] = "telegram_link"

            bot_username = context.bot.username
            link = f"https://t.me/{bot_username}?start={unique_id}"
            
            await message.reply_text(
                f"✅ <b>لینک رمزدار شما آماده شد!</b>\n\n🔐 <b>رمز عبور:</b> <code>{password}</code>\n🔗 <b>لینک:</b>\n{link}",
                parse_mode="HTML",
                reply_markup=get_back_keyboard()
            )
        return

    mode = user_mode.get(user_id, "telegram_link")

    if mode == "upload":
        media = message.document or message.video or message.audio or message.voice or (message.photo[-1] if message.photo else None)
        if not media:
            await message.reply_text("لطفاً یک فایل معتبر بفرستید.")
            return
        status = await message.reply_text("⏳ در حال دریافت فایل...")
        file_path = await (await media.get_file()).download_to_drive()
        user_files[user_id] = file_path
        await status.edit_text("📂 فایل دریافت شد. سرور را انتخاب کنید:", reply_markup=get_server_choice_keyboard())
        return

    # ۳. دریافت محتوای جدید برای ساخت لینک
    content_data = {}
    if message.text:
        content_data = {"type": "text", "content": message.text}
    elif message.photo:
        content_data = {"type": "photo", "content": message.photo[-1].file_id, "caption": message.caption or ""}
    elif message.document:
        content_data = {"type": "document", "content": message.document.file_id, "caption": message.caption or ""}
    elif message.video:
        content_data = {"type": "video", "content": message.video.file_id, "caption": message.caption or ""}
    elif message.audio:
        content_data = {"type": "audio", "content": message.audio.file_id, "caption": message.caption or ""}
    elif message.voice:
        content_data = {"type": "voice", "content": message.voice.file_id}
    else:
        await message.reply_text("❌ این فرمت پشتیبانی نمی‌شود.")
        return

    user_pending[user_id] = content_data
    await message.reply_text(
        "⚙️ محتوای شما دریافت شد. لطفاً نوع لینک را انتخاب کنید:",
        reply_markup=get_options_keyboard()
    )

def main():
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(button_handler))
    app.add_handler(MessageHandler(
        filters.TEXT | filters.PHOTO | filters.VIDEO | filters.Document.ALL | filters.AUDIO | filters.VOICE,
        handle_incoming_content
    ))
    print("ربات با قابلیت لینک رمزدار و یک‌بار مصرف روشن شد...")
    app.run_polling()

if __name__ == '__main__':
    main()
    
