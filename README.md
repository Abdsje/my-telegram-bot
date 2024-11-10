import os
import logging
from telegram import Update, ChatPermissions, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackQueryHandler, CallbackContext
from datetime import timedelta

# إعدادات البوت
TOKEN = os.getenv("TOKEN")  # تأكد من إضافة التوكن الخاص بك كمتغير بيئي
OWNER_ID = os.getenv("OWNER_ID")  # معرف صاحب البوت

# إعدادات تسجيل الأخطاء
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
                    level=logging.INFO)
logger = logging.getLogger(__name__)

# دالة لتنفيذ العقوبة (كتم المستخدم)
def penalize_user(update: Update, context: CallbackContext):
    user = update.message.from_user  # المستخدم الذي أرسل الرسالة
    chat_id = update.message.chat_id  # معرف القناة أو المجموعة
    message_id = update.message.message_id
    try:
        # حذف الرسالة التي تحتوي على رابط
        update.message.delete()
        # كتم المستخدم لمدة يوم واحد كعقوبة
        context.bot.restrict_chat_member(
            chat_id=chat_id,
            user_id=user.id,        permissions=ChatPermissions(can_send_messages=False),
            until_date=update.message.date + timedelta(days=1)
        )     
        # إرسال رسالة تحذيرية
        context.bot.send_message(
            chat_id=chat_id,
            text=f"تم كتم المستخدم {user.first_name} لمدة 24 ساعة بسبب إرسال روابط.",
            reply_to_message_id=message_id
        )
        print("تم كتم المستخدم الذي أرسل رابط.")
    except Exception as e:
        print(f"حدث خطأ أثناء محاولة تنفيذ العقوبة: {e}")
# دالة للرد على الأمر /start
def start_command(update: Update, context: CallbackContext):
    chat_id = update.message.chat_id
    response = (
        "مرحباً! أنا بوت إدارة المجموعة أو القناة، وسأساعد في حذف الروابط وتطبيق العقوبات على المخالفين.\n\n"
        "لإضافتي إلى قناتك أو مجموعتك:\n"
        "1. تأكد من أنني مشرف في القناة أو المجموعة.\n"
        "2. قم بتفعيل صلاحيات إدارة الرسائل حتى أتمكن من حذف الرسائل وتطبيق العقوبات.\n\n"
        "بمجرد أن أكون مشرفًا، سأبدأ في مراقبة الرسائل وحذف الروابط عند وجودها."
    )    context.bot.send_message(chat_id=chat_id, text=response)
# دالة لعرض لوحة الإعدادات
def settings_command(update: Update, context: CallbackContext):
    keyboard = [
        [InlineKeyboardButton("عرض المحظورين", callback_data='view_banned')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    update.message.reply_text('مرحبًا، اختر من الخيارات التالية:', reply_markup=reply_markup)
# دالة لمعالجة الخيارات
def button_callback(update: Update, context: CallbackContext):
    query = update.callback_query
    if query.data == 'view_banned':
        # عرض الأشخاص الذين تم حظرهم (أو المحظورين) 
        banned_users = get_banned_users()  # استخدم دالة لجلب المحظورين من قاعدة البيانات أو الذاكرة
        banned_list = "\n".join(banned_users) if banned_users else "لا يوجد محظورين حالياً."
        query.edit_message_text(f"المستخدمون المحظورون:\n{banned_list}")
# دالة لاسترجاع المحظورين (يجب أن يتم تنفيذها بناءً على طريقة حفظ المحظورين)
def get_banned_users():
    # هنا يجب أن تضع طريقة لاسترجاع قائمة المستخدمين المحظورين (مثلاً من قاعدة بيانات أو قائمة في الذاكرة)
    return ["user1", "user2"]  # مثال
# إعداد البوت
updater = Updater(TOKEN, use_context=True)
dp = updater.dispatcher
# إضافة الأوامر إلى البوت
dp.add_handler(CommandHandler("start", start_command))
dp.add_handler(CommandHandler("settings", settings_command))
dp.add_handler(MessageHandler(Filters.text & ~Filters.command & Filters.entity("url"), penalize_user))
dp.add_handler(CallbackQueryHandler(button_callback))
# بدء تشغيل البوت
updater.start_polling()
updater.idle()
