from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes

TOKEN = "BOT_TOKEN_BURAYA"

# 👇 SENİN ADMIN ID
ADMIN_ID = 7793219370

IBAN = "TR97 0001 0002 5593 8548 1650 01"
IBAN_NAME = "Mustafa Ekrem Birdağ"
SUPPORT = "@efes65lik"

# 📦 Ürün & stok
products = {
    "urun1": {
        "name": "Dijital Ürün 1",
        "price": 100,
        "stock": ["CODE-111", "CODE-222"]
    },
    "urun2": {
        "name": "Dijital Ürün 2",
        "price": 50,
        "stock": ["CODE-333"]
    }
}

user_orders = {}

# /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("🛍 Ürünler", callback_data="products")],
        [InlineKeyboardButton("📦 Siparişlerim", callback_data="orders")],
        [InlineKeyboardButton("🆘 Destek", url=f"https://t.me/{SUPPORT.replace('@','')}")]
    ]
    await update.message.reply_text("Hoş geldin 👋", reply_markup=InlineKeyboardMarkup(keyboard))


# Menü
async def menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    if query.data == "products":
        text = "🛍 Ürünler:\n\n"
        keyboard = []

        for key, p in products.items():
            text += f"{p['name']} - {p['price']} TL\n"
            keyboard.append([InlineKeyboardButton(p["name"], callback_data=f"buy_{key}")])

        await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard))

    elif query.data == "orders":
        uid = query.from_user.id
        order = user_orders.get(uid, "Sipariş yok")
        await query.edit_message_text(f"📦 Siparişin: {order}")

    elif query.data.startswith("buy_"):
        key = query.data.split("_")[1]
        user_orders[query.from_user.id] = key

        product = products[key]

        await query.message.reply_text(
            f"💰 Ödeme Bilgileri:\n\n"
            f"IBAN: {IBAN}\n"
            f"İsim: {IBAN_NAME}\n\n"
            f"Ücret: {product['price']} TL\n\n"
            "📩 Dekontu buraya gönder"
        )


# 📩 Dekont
async def receipt(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.message.from_user.id

    if uid not in user_orders:
        await update.message.reply_text("Önce ürün seç 😄")
        return

    product_key = user_orders[uid]
    product = products[product_key]

    await context.bot.send_message(
        chat_id=ADMIN_ID,
        text=f"📦 Yeni sipariş\nUser: {uid}\nÜrün: {product['name']}"
    )

    await update.message.reply_text("📩 Dekont alındı, admin onayı bekleniyor.")


# ✅ Admin onay
async def approve(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.from_user.id != ADMIN_ID:
        return

    try:
        uid = int(context.args[0])
    except:
        await update.message.reply_text("Kullanım: /onay user_id")
        return

    if uid not in user_orders:
        await update.message.reply_text("Sipariş yok")
        return

    key = user_orders[uid]
    product = products[key]

    if product["stock"]:
        code = product["stock"].pop(0)

        await context.bot.send_message(
            chat_id=uid,
            text=f"✅ Onaylandı!\nKodun: {code}"
        )

        await update.message.reply_text("Onaylandı ✔")
    else:
        await update.message.reply_text("Stok yok ❌")


# bot
app = Application.builder().token(TOKEN).build()

app.add_handler(CommandHandler("start", start))
app.add_handler(CallbackQueryHandler(menu))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, receipt))
app.add_handler(CommandHandler("onay", approve))

app.run_polling()
