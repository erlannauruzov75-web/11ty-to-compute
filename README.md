"""
ALP SIGNAL — Profit Edge (Rewritten)
- 9s analysis (1 sample/second)
- Shows assets -> choose -> shows timeframes -> choose -> runs 9s analysis -> sends BUY or SELL image
- Uses Twelve Data for prices. ENV: TELEGRAM_TOKEN, TWELVE_API_KEY
"""

import os
import asyncio
import requests
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, ContextTypes
from PIL import Image, ImageDraw, ImageFont, ImageFilter
import io
import statistics

# ---- CONFIG ----
BOT_TOKEN = os.getenv("8006148136:AAF6C3WR9XX6TwmEZ4g8uNaSf86Z6v3Ui5k", "PASTE_YOUR_TOKEN_HERE")
TWELVE_API_KEY = os.getenv("ebce69ce5a8f48dfb5dfd2bb6e7e0744", "PASTE_YOUR_TWELVE_DATA_KEY_HERE")

ASSETS = [
    "EUR/USD", "GBP/USD", "CHF/JPY", "AUD/CHF", "CHF/NOK",
    "AUD/CAD", "EUR/RUB", "AUD/USD", "AUD/NZD", "USD/JPY"
]

TIMEFRAMES = [
    "S5","S10","S15","S20","S25","S30","S40","S50","S55",
    "M1","M2","M3","M4","M5"
]

# Image sizes
BANNER_W, BANNER_H = 900, 150
CARD_W, CARD_H = 900, 450
FINAL_W, FINAL_H = 900, 600

# ---- Utility: fetch price from TwelveData ----
def fetch_price(symbol: str):
    sym = symbol.replace("/", "")
    urls = [
        f"https://api.twelvedata.com/price?symbol={sym}&apikey={TWELVE_API_KEY}",
        f"https://api.twelvedata.com/price?symbol={symbol}&apikey={TWELVE_API_KEY}"
    ]
    for url in urls:
        try:
            r = requests.get(url, timeout=8)
            j = r.json()
            if isinstance(j, dict) and "price" in j:
                return float(j["price"])
        except Exception:
            continue
    return None

# ---- Indicators ----
def ema_series(data, period):
    if not data:
        return []
    if len(data) < period:
        avg = sum(data) / len(data)
        return [avg] * len(data)
    k = 2 / (period + 1)
    ema_vals = []
    sma = sum(data[:period]) / period
    ema_vals = [sma]
    for price in data[period:]:
        ema_vals.append(price * k + ema_vals[-1] * (1 - k))
    pad = len(data) - len(ema_vals)
    return [ema_vals[0]] * pad + ema_vals

def current_ema(data, period):
    s = ema_series(data, period)
    return s[-1] if s else None

def macd_value(data):
    e12 = current_ema(data, 12)
    e26 = current_ema(data, 26)
    if e12 is None or e26 is None:
        return 0.0
    return e12 - e26

def compute_rsi(data, period=14):
    if len(data) < 2:
        return 50.0
    deltas = [data[i+1] - data[i] for i in range(len(data)-1)]
    gains = [d if d>0 else 0 for d in deltas]
    losses = [-d if d<0 else 0 for d in deltas]
    if len(deltas) < period:
        avg_gain = sum(gains) / max(1, len(gains))
        avg_loss = sum(losses) / max(1, len(losses))
    else:
        avg_gain = sum(gains[:period]) / period
        avg_loss = sum(losses[:period]) / period
        for i in range(period, len(deltas)):
            avg_gain = (avg_gain * (period - 1) + gains[i]) / period
            avg_loss = (avg_loss * (period - 1) + losses[i]) / period
    if avg_loss == 0:
        return 100.0
    rs = avg_gain / avg_loss
    return 100 - (100 / (1 + rs))

# ---- Decision logic (strict + fallback) ----
def decide_signal(prices):
    # prices: list of floats (chronological)
    if not prices or len(prices) < 2:
        return None
    ema5 = current_ema(prices, 5)
    ema20 = current_ema(prices, 20)
    macd = macd_value(prices)
    rsi = compute_rsi(prices, period=14 if len(prices) >= 15 else max(3, len(prices)-1))
    volatility = statistics.pstdev(prices) if len(prices) > 1 else 0.0

    # Strict optimized rule
    if ema5 is not None and ema20 is not None:
        if ema5 > ema20 and macd > 0 and rsi < 75 and volatility > 0:
            return "BUY"
        if ema5 < ema20 and macd < 0 and rsi > 25 and volatility > 0:
            return "SELL"

    # Fallback: price trend across samples
    if prices[-1] > prices[0]:
        return "BUY"
    else:
        return "SELL"

# ---- Graphics: banner + card compose ----
def create_banner():
    img = Image.new("RGBA", (BANNER_W, BANNER_H), (20,20,20,255))
    draw = ImageDraw.Draw(img)
    # gold gradient
    for y in range(BANNER_H):
        t = y / (BANNER_H - 1)
        r = int(60 + 180 * (1 - t))
        g = int(45 + 160 * (1 - t))
        b = int(20 + 120 * (1 - t))
        draw.line([(0,y),(BANNER_W,y)], fill=(r,g,b,255))
    # overlay
    overlay = Image.new("RGBA", (BANNER_W, BANNER_H), (0,0,0,120))
    img = Image.alpha_composite(img, overlay)
    draw = ImageDraw.Draw(img)
    # chart line
    import math
    pts = [(30 + i*(BANNER_W-60)/20, 70 + math.sin(i/2.8)*22) for i in range(21)]
    draw.line(pts, fill=(255,215,0,200), width=3)
    try:
        fp = "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf"
        f_icon = ImageFont.truetype(fp, 36)
        f_main = ImageFont.truetype(fp, 28)
    except:
        f_icon = ImageFont.load_default()
        f_main = ImageFont.load_default()
    draw.text((28,55), "📈", font=f_icon, fill=(255,223,80,255))
    draw.text((75,56), "Profit Edge  |  Real Market Analyzer", font=f_main, fill=(255,223,80,255))
    return img.convert("RGB")

def create_card(signal, symbol):
    bg = (20,120,60) if signal=="BUY" else (170,30,30)
    img = Image.new("RGB", (CARD_W,CARD_H), bg)
    draw = ImageDraw.Draw(img)
    try:
        fp = "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf"
        f_big = ImageFont.truetype(fp, 110)
        f_sym = ImageFont.truetype(fp, 50)
        f_small = ImageFont.truetype(fp, 22)
    except:
        f_big = ImageFont.load_default()
        f_sym = ImageFont.load_default()
        f_small = ImageFont.load_default()
    text = "BUY  🔺" if signal=="BUY" else "SELL  🔻"
    w,h = draw.textsize(text, font=f_big)
    draw.text(((CARD_W-w)/2, 80), text, font=f_big, fill=(255,255,255))
    ws,hs = draw.textsize(symbol, font=f_sym)
    draw.text(((CARD_W-ws)/2, 280), symbol, font=f_sym, fill=(255,255,255))
    draw.text((CARD_W-260, CARD_H-40), "ALP SIGNAL — Profit Edge", font=f_small, fill=(255,255,255))
    return img

def compose_image(banner, card):
    final = Image.new("RGB", (FINAL_W, FINAL_H), (0,0,0))
    final.paste(banner.resize((BANNER_W,BANNER_H)), (0,0))
    final.paste(card.resize((CARD_W,CARD_H)), (0,BANNER_H))
    bio = io.BytesIO()
    final.save(bio, format="PNG")
    bio.seek(0)
    return bio

# ---- Bot handlers ----
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    kb=[]
    row=[]
    for i,s in enumerate(ASSETS,1):
        row.append(InlineKeyboardButton(s, callback_data=f"asset|{s}"))
        if i%2==0:
            kb.append(row); row=[]
    if row: kb.append(row)
    await update.message.reply_text("ALP SIGNAL — aktivni tanlang:", reply_markup=InlineKeyboardMarkup(kb))

async def callback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    data = query.data

    if data.startswith("asset|"):
        asset = data.split("|",1)[1]
        # save selection in user_data
        context.user_data["asset"] = asset
        # show timeframe buttons
        kb=[]
        row=[]
        for i,tf in enumerate(TIMEFRAMES,1):
            row.append(InlineKeyboardButton(tf, callback_data=f"tf|{tf}"))
            if i%3==0:
                kb.append(row); row=[]
        if row: kb.append(row)
        await query.edit_message_text(f"Tanlandi: <b>{asset}</b>\nTimeframe tanlang:", parse_mode="HTML", reply_markup=InlineKeyboardMarkup(kb))
        return

    if data.startswith("tf|"):
        tf = data.split("|",1)[1]
        asset = context.user_data.get("asset")
        if not asset:
            await query.edit_message_text("Avval aktivni tanlang /start bilan.")
            return
        # Acknowledge and perform 9s analysis now
        await query.edit_message_text(f"{asset} — {tf} uchun 9 soniyalik tahlil boshlandi...")
        prices=[]
        # collect 9 prices (1s apart)
        for i in range(9):
            p = fetch_price(asset)
            if p is not None:
                prices.append(p)
            # small safety sleep even if fetch was instant
            await asyncio.sleep(1)

        if len(prices) < 2:
            # If TwelveData limit or network error, fallback to try once more quickly
            await query.message.reply_text("Narx olinmadi — API limit yoki tarmoq. Keyinroq qayta urin.")
            return

        # decide signal (always returns BUY or SELL)
        signal = decide_signal(prices)

        # create and send image (only image, no extra text)
        banner = create_banner()
        card = create_card(signal, asset)
        bio = compose_image(banner, card)
        await query.message.reply_photo(photo=bio)
        return

# ---- Main ----
def main():
    if BOT_TOKEN.startswith("PASTE_") or TWELVE_API_KEY.startswith("PASTE_"):
        print("Set TELEGRAM_TOKEN and TWELVE_API_KEY environment variables.")
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(callback_handler))
    print("Bot started...")
    app.run_polling()

if __name__ == "__main__":
    main()
