from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, ContextTypes, filters
from telegram.error import TelegramError
import re
from collections import defaultdict
import time
import logging
import json
import os

# تنظیم لاگ برای عیب‌یابی
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# تنظیمات
BOT_TOKEN = "7567114327:AAGvYzNo-jY5kEAwovv3MrudYuwdY1S3ahE"  # توکن ربات
SPAM_THRESHOLD = 5  # تعداد پیام در بازه زمانی برای تشخیص اسپم
SPAM_TIME_WINDOW = 10  # بازه زمانی (ثانیه)
WARNING_LIMIT = 5  # حداکثر اخطار قبل از بن

# ذخیره اطلاعات
user_warnings = defaultdict(float)  # ذخیره اخطارهای کاربران
user_messages = defaultdict(list)  # ذخیره زمان پیام‌ها
custom_admins = set()  # لیست ادمین‌های سفارشی

async def is_admin(update: Update, context: ContextTypes.DEFAULT_TYPE) -> bool:
    """چک می‌کنه که کاربر ادمینه یا نه"""
    message = update.effective_message
    if not message:
        logger.error("No effective message found in update")
        return False
    user_id = message.from_user.id
    chat_id = message.chat.id
    try:
        member = await context.bot.get_chat_member(chat_id, user_id)
        is_admin = member.status in ["administrator", "creator"] or user_id in custom_admins
        logger.info(f"User {user_id} admin check: {is_admin} in chat {chat_id}")
        return is_admin
    except TelegramError as e:
        logger.error(f"Error checking admin status for user {user_id}: {e}")
        await message.reply_text("من دسترسی لازم برای مدیریت گروه ندارم. لطفاً منو ادمین کنید و دسترسی‌های لازم رو بدید.")
        return False

async def is_user_in_chat(chat_id: int, user_id: int, context: ContextTypes.DEFAULT_TYPE) -> bool:
    """چک می‌کنه که کاربر توی گروهه"""
    try:
        await context.bot.get_chat_member(chat_id, user_id)
        return True
    except TelegramError:
        logger.info(f"User {user_id} not found in chat {chat_id}")
        return False

async def is_user_restricted(chat_id: int, user_id: int, context: ContextTypes.DEFAULT_TYPE) -> bool:
    """چک می‌کنه که کاربر میوته"""
    try:
        member = await context.bot.get_chat_member(chat_id, user_id)
        return not member.can_send_messages
    except TelegramError:
        return False

async def check_spam(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """تشخیص اسپم و میوت برای 10 دقیقه"""
    message = update.effective_message
    if not message:
        logger.error("No effective message in check_spam")
        return
    user_id = message.from_user.id
    chat_id = message.chat.id
    current_time = time.time()

    user_messages[user_id].append(current_time)
    user_messages[user_id] = [t for t in user_messages[user_id] if current_time - t < SPAM_TIME_WINDOW]

    if len(user_messages[user_id]) > SPAM_THRESHOLD:
        try:
            await context.bot.restrict_chat_member(
                chat_id, user_id, until_date=int(time.time()) + 600, permissions={"can_send_messages": False}
            )
            await message.reply_text(
                f"{message.from_user.mention_html()}\nبه دلیل اسپم برای 10 دقیقه میوت شد.\nخموش باش بنده حقیر",
                parse_mode="HTML"
            )
            user_messages[user_id].clear()
            logger.info(f"User {user_id} muted for spam in chat {chat_id}")
        except TelegramError as e:
            logger.error(f"Error restricting user {user_id} for spam: {e}")
            await message.reply_text("خطا در میوت کردن. لطفاً دسترسی‌های ربات رو چک کنید.")

async def check_links(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """تشخیص لینک و دادن اخطار"""
    message = update.effective_message
    if not message:
        logger.error("No effective message in check_links")
        return
    message_text = message.text or ""
    if re.search(r"(t\.me/|telegram\.me/|@[\w\d_]+)", message_text):
        user_id = message.from_user.id
        user_warnings[user_id] += 1
        await message.reply_text(
            f"{message.from_user.mention_html()}\nبه دلیل ارسال لینک اخطار گرفت. اخطارها: {user_warnings[user_id]}/{WARNING_LIMIT}",
            parse_mode="HTML"
        )
        logger.info(f"User {user_id} warned for link. Warnings: {user_warnings[user_id]}")
        if user_warnings[user_id] >= WARNING_LIMIT:
            try:
                await context.bot.ban_chat_member(message.chat.id, user_id)
                await message.reply_text(
                    f"{message.from_user.mention_html()}\nبه دلیل رسیدن به حد اخطار بن شد.",
                    parse_mode="HTML"
                )
                user_warnings[user_id] = 0
                logger.info(f"User {user_id} banned for exceeding warnings in chat {message.chat.id}")
            except TelegramError as e:
                logger.error(f"Error banning user {user_id}: {e}")
                await message.reply_text("خطا در بن کردن. لطفاً دسترسی‌های ربات رو چک کنید.")

async def unwarn_user(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """لغو اخطار کاربر"""
    message = update.effective_message
    if not message or not message.reply_to_message:
        await message.reply_text("لطفاً روی پیام کاربر ریپلای کنید.")
        return
    if not await is_admin(update, context):
        await message.reply_text("فقط ادمین‌ها می‌تونن اخطار لغو کنن.")
        return

    target_user = message.reply_to_message.from_user
    target_user_id = target_user.id
    chat_id = message.chat.id

    if target_user_id not in user_warnings or user_warnings[target_user_id] == 0:
        await message.reply_text(f"{target_user.mention_html()}\nهیچ اخطاری نداره!", parse_mode="HTML")
        return

    user_warnings[target_user_id] -= 1
    await message.reply_text(
        f"{target_user.mention_html()}\nیه اخطار لغو شد. اخطارهای باقی‌مونده: {user_warnings[target_user_id]}/{WARNING_LIMIT}",
        parse_mode="HTML"
    )
    logger.info(f"Warning removed for user {target_user_id}. Current warnings: {user_warnings[target_user_id]}")

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """نمایش لیست دستورات ربات"""
    help_text = (
        "📋 لیست دستورات ربات:\n\n"
        "👮‍♂️ دستورات ادمین (با ریپلای روی پیام کاربر):\n"
        "- اخطار: دادن اخطار به کاربر\n"
        "- لغو اخطار: کم کردن یک اخطار از کاربر\n"
        "- خفه <زمان>: میوت کاربر (مثال: خفه 10m یا خفه 2h)\n"
        "- نفس بکش: رفع میوت کاربر\n"
        "- سیک: بن کردن کاربر\n"
        "- بیا پایین: حذف ادمین از گروه\n"
        "- ارتقا به خایمال رادمان: اضافه کردن کاربر به ادمین‌های سفارشی ربات\n\n"
        "🤖 سایر:\n"
        "- ربات: تست ربات (جواب متفاوته برای ادمین و غیرادمین!)\n"
        "- /help: نمایش این راهنما"
    )
    await update.effective_message.reply_text(help_text)

async def handle_admin_commands(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """مدیریت دستورات ادمین"""
    message = update.effective_message
    if not message:
        logger.error("No effective message in handle_admin_commands")
        return

    has_reply = message.reply_to_message is not None
    is_admin_user = await is_admin(update, context)
    logger.info(f"Command received. Has reply: {has_reply}, Is admin: {is_admin_user}")

    if not has_reply or not is_admin_user:
        return

    target_user = message.reply_to_message.from_user
    chat_id = message.chat.id
    target_user_id = target_user.id

    if not await is_user_in_chat(chat_id, target_user_id, context):
        await message.reply_text(f"{target_user.mention_html()}\nزیر دستت فرار کرده", parse_mode="HTML")
        return

    temp_update = Update(update.update_id, message.reply_to_message)
    target_is_admin = await is_admin(temp_update, context)

    command = message.text.lower().strip()
    logger.info(f"Processing command: {command} for user {target_user_id}")

    if command == "بیا پایین" and target_is_admin:
        try:
            await context.bot.promote_chat_member(
                chat_id, target_user_id,
                can_manage_chat=False, can_delete_messages=False,
                can_restrict_members=False, can_promote_members=False
            )
            if target_user_id in custom_admins:
                custom_admins.remove(target_user_id)
            await message.reply_text(f"{target_user.mention_html()}\nبیا پایین سرمون درد گرفت", parse_mode="HTML")
            logger.info(f"User {target_user_id} demoted in chat {chat_id}")
        except TelegramError as e:
            logger.error(f"Error demoting user {target_user_id}: {e}")
            await message.reply_text(f"خطا در حذف ادمین: {e}", parse_mode="HTML")
        return

    if command == "نفس بکش":
        if await is_user_restricted(chat_id, target_user_id, context):
            try:
                await context.bot.restrict_chat_member(
                    chat_id, target_user_id, permissions={"can_send_messages": True}
                )
                await message.reply_text(f"{target_user.mention_html()}\nدهنشو باز کردم", parse_mode="HTML")
                logger.info(f"User {target_user_id} unmuted in chat {chat_id}")
            except TelegramError as e:
                logger.error(f"Error unrestricting user {target_user_id}: {e}")
                await message.reply_text(f"خطا در رفع میوت: {e}", parse_mode="HTML")
        else:
            await message.reply_text(f"{target_user.mention_html()}\nدهنش بازه", parse_mode="HTML")
        return

    if command == "اخطار":
        user_warnings[target_user_id] += 1
        await message.reply_text(
            f"{target_user.mention_html()}\nاخطار گرفت. اخطارها: {user_warnings[target_user_id]}/{WARNING_LIMIT}",
            parse_mode="HTML"
        )
        logger.info(f"User {target_user_id} warned. Total warnings: {user_warnings[target_user_id]}")
        if user_warnings[target_user_id] >= WARNING_LIMIT:
            try:
                await context.bot.ban_chat_member(chat_id, target_user_id)
                await message.reply_text(
                    f"{target_user.mention_html()}\nبه دلیل رسیدن به حد اخطار بن شد.",
                    parse_mode="HTML"
                )
                user_warnings[target_user_id] = 0
                logger.info(f"User {target_user_id} banned for exceeding warnings")
            except TelegramError as e:
                logger.error(f"Error banning user {target_user_id}: {e}")
                await message.reply_text(f"خطا در بن کردن: {e}", parse_mode="HTML")

    elif command == "لغو اخطار":
        await unwarn_user(update, context)

    elif command == "ارتقا به خایمال رادمان":
        custom_admins.add(target_user_id)
        await message.reply_text(f"{target_user.mention_html()}\nبه خایمال رادمان ارتقا یافت!", parse_mode="HTML")
        logger.info(f"User {target_user_id} promoted to custom admin")

    elif command == "سیک":
        try:
            await context.bot.ban_chat_member(chat_id, target_user_id)
            await message.reply_text(f"{target_user.mention_html()}\nبن شد!", parse_mode="HTML")
            logger.info(f"User {target_user_id} banned in chat {chat_id}")
        except TelegramError as e:
            logger.error(f"Error banning user {target_user_id}: {e}")
            await message.reply_text(f"خطا در بن کردن: {e}", parse_mode="HTML")

    elif command.startswith("خفه"):
        try:
            parts = command.split()
            if len(parts) != 2:
                await message.reply_text(f"فرمت دستور اشتباهه: خفه 10m یا خفه 2h")
                return
            time_str = parts[1]
            match = re.match(r"(\d+)([mh])", time_str)
            if not match:
                await message.reply_text(f"فرمت زمان اشتباهه. مثال: خفه 10m")
                return
            amount, unit = int(match.group(1)), match.group(2)
            seconds = amount * 60 if unit == "m" else amount * 3600
            await context.bot.restrict_chat_member(
                chat_id, target_user_id, until_date=int(time.time()) + seconds,
                permissions={"can_send_messages": False}
            )
            unit_str = "دقیقه" if unit == "m" else "ساعت"
            await message.reply_text(
                f"{target_user.mention_html()}\nبرای {amount} {unit_str} میوت شد.\nخفش کردم",
                parse_mode="HTML"
            )
            logger.info(f"User {target_user_id} muted for {seconds} seconds")
        except TelegramError as e:
            logger.error(f"Error muting user {target_user_id}: {e}")
            await message.reply_text(f"خطا در میوت کردن: {e}", parse_mode="HTML")

async def handle_robot_call(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """مدیریت وقتی کسی می‌گه 'ربات'"""
    message = update.effective_message
    if not message:
        logger.error("No effective message in handle_robot_call")
        return
    if message.chat.type in ["group", "supergroup"]:
        if message.text.lower().strip() == "ربات":
            if await is_admin(update, context):
                await message.reply_text("جون")
            else:
                await message.reply_text("کون")
            logger.info(f"User {message.from_user.id} called 'ربات' in chat {message.chat.id}")

async def message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """مدیریت تمام پیام‌ها"""
    message = update.effective_message
    if not message:
        logger.error("No effective message in message_handler")
        return
    if message.chat.type in ["group", "supergroup"]:
        if message.text and not message.text.startswith("/"):
            await check_spam(update, context)
            await check_links(update, context)
            await handle_admin_commands(update, context)
            await handle_robot_call(update, context)

def main():
    app = Application.builder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, message_handler))
    logger.info("Bot started")
    app.run_polling()

if __name__ == "__main__":
    main()
