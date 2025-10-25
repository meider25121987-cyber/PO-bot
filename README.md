import asyncio
import aiohttp
from telegram import Update, KeyboardButton, ReplyKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes

BOT_TOKEN = "ТВОЙ_ТОКЕН_ИЗ_BotFather"

user_selection = {}

# === АНАЛИЗ ===
async def analyze_pair(pair: str, tf: str):
    url = f"https://api.binance.com/api/v3/ticker/price?symbol={pair.replace('/', '')}"
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            data = await resp.json()
            price = float(data['price'])
            return f"📊 {pair}\n⏱️ Таймфрейм: {tf}\n💰 Цена: {price:.5f}\n📈 Сигнал: наблюдай за изменением тренда."

# === СТАРТ ===
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [KeyboardButton("EUR/USD"), KeyboardButton("GBP/USD")],
        [KeyboardButton("USD/JPY"), KeyboardButton("AUD/USD")],
        [KeyboardButton("BTC/USDT"), KeyboardButton("ETH/USDT")],
        [KeyboardButton("DOGE/USDT"), KeyboardButton("SOL/USDT")]
    ]
    markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
    await update.message.reply_text("👋 Привет! Выбери валютную пару или крипту:", reply_markup=markup)

# === ВЫБОР ПАРЫ ===
async def handle_pair(update: Update, context: ContextTypes.DEFAULT_TYPE):
    pair = update.message.text.strip().upper()
    if '/' not in pair:
        await update.message.reply_text("⚠️ Укажи валютную пару в формате BTC/USDT.")
        return

    user_selection[update.effective_user.id] = {"pair": pair}
    keyboard = [
        [KeyboardButton("1 мин"), KeyboardButton("2 мин"), KeyboardButton("3 мин")],
        [KeyboardButton("5 мин"), KeyboardButton("10 мин"), KeyboardButton("1 час")],
        [KeyboardButton("🔙 Назад")]
    ]
    markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
    await update.message.reply_text(f"✅ Выбрано: {pair}\nТеперь выбери таймфрейм:", reply_markup=markup)

# === ВЫБОР ТАЙМФРЕЙМА ===
async def handle_timeframe(update: Update, context: ContextTypes.DEFAULT_TYPE):
    tf = update.message.text.strip()
    user_id = update.effective_user.id

    if tf == "🔙 Назад":
        await start(update, context)
        return

    if user_id not in user_selection or "pair" not in user_selection[user_id]:
        await update.message.reply_text("Сначала выбери валютную пару!")
        return

    pair = user_selection[user_id]["pair"]
    await update.message.reply_text(f"⏳ Анализ {pair} ({tf})...")

    try:
        result = await analyze_pair(pair, tf)
        await update.message.reply_text(result)
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка анализа: {e}")

# === ЗАПУСК ===
async def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    # теперь фильтры работают для любых вариантов текста
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_router))
    print("✅ Бот запущен. Открой Telegram и нажми Start.")
    await app.run_polling()

# === УМНЫЙ РОУТЕР ===
async def handle_router(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.strip().lower()
    if "/" in text:  # валютная пара
        await handle_pair(update, context)
    elif "мин" in text or "час" in text or "назад" in text:
        await handle_timeframe(update, context)
    else:
        await update.message.reply_text("Выбери валютную пару или таймфрейм с кнопок.")

if __name__ == "__main__":
    asyncio.run(main())
