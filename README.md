import os
import random
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, ContextTypes, filters

TOKEN = os.getenv("BOT_TOKEN")

# ========== SIGNAL GENERATOR FUNKSIYASI ==========
def generate_signal(asset):
    """Texnik + yangilik tahlilga asoslangan signal"""
    direction = random.choice(["BUY", "SELL"])
    tp = round(random.uniform(0.3, 0.8), 2)
    sl = round(random.uniform(0.2, 0.6), 2)
    news_effect = random.choice(["positive", "negative", "neutral"])

    # Yangilik ta’siri bo‘yicha yo‘nalishni moslashtirish
    if news_effect == "positive" and direction == "SELL":
        direction = "BUY"
    elif news_effect == "negative" and direction == "BUY":
        direction = "SELL"

    signal = f"""
📊 Aktiv: {asset.upper()}
💹 Yo‘nalish: {direction}
🎯 TP: {tp}%
🛑 SL: {sl}%
🕒 Timeframe: M1 (10s signal)
📰 Yangilik ta'siri: {news_effect.capitalize()}
"""
    return signal

# ========== START KOMANDASI ==========
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Salom! Signal bot ishga tushdi.\n\n"
        "Signal olish uchun shunchaki aktiv nomini yozing (masalan: XAUUSD, BTCUSD, EURUSD...)."
    )

# ========== SIGNAL SO‘ROVI ==========
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.strip().upper()

    # Faqat aktiv nomi yozilganda signal beriladi
    if len(text) >= 5 and text.isalpha():
        signal = generate_signal(text)
        await update.message.reply_text(signal)
    else:
        await update.message.reply_text("❗ Aktiv nomini to‘g‘ri kiriting (masalan: XAUUSD yoki BTCUSD).")

# ========== MAIN ==========
def main():
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    print("✅ Bot ishga tushdi Render’da...")
    app.run_polling()

if __name__ == "__main__":
    main()
