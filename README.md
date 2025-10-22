# quotex-signal-bot
import os
import time
import requests
import pandas as pd
from telegram import Bot
from dotenv import load_dotenv

load_dotenv()

TELEGRAM_TOKEN = os.getenv("TELEGRAM_TOKEN")
CHAT_ID = os.getenv("CHAT_ID")
SYMBOL = os.getenv("SYMBOL", "EURUSD=X")
INTERVAL = os.getenv("INTERVAL", "1m")
FAST_WINDOW = int(os.getenv("FAST_WINDOW", "7"))
SLOW_WINDOW = int(os.getenv("SLOW_WINDOW", "21"))
POLL_SECONDS = int(os.getenv("POLL_SECONDS", "60"))

bot = Bot(token=)

def fetch_price(symbol=SYMBOL):
    url = f"https://query1.finance.yahoo.com/v8/finance/chart/{symbol}?interval={INTERVAL}&range=1d"
    r = requests.get(url, timeout=10)
    r.raise_for_status()
    data = r.json()
    closes = data["chart"]["result"][0]["indicators"]["quote"][0]["close"]
    df = pd.DataFrame({"close": closes})
    return df

def compute_signal(df):
    df = df.copy()
    df["sma_fast"] = df["close"].rolling(FAST_WINDOW, min_periods=1).mean()
    df["sma_slow"] = df["close"].rolling(SLOW_WINDOW, min_periods=1).mean()
    if len(df) < 2:
        return "HOLD", None
    prev = df.iloc[-2]
    last = df.iloc[-1]
    if prev["sma_fast"] <= prev["sma_slow"] and last["sma_fast"] > last["sma_slow"]:
        return "UP ⬆️", last["close"]
    if prev["sma_fast"] >= prev["sma_slow"] and last["sma_fast"] < last["sma_slow"]:
        return "DOWN ⬇️", last["close"]
    return "HOLD ⚪", last["close"]

def send_message(text):
    try:
        bot.send_message(chat_id=CHAT_ID, text=text, parse_mode="HTML")
    except Exception as e:
        print("Failed to send message:", e)

def format_signal_message(signal, price):
    t = time.strftime("%Y-%m-%d %H:%M:%S UTC", time.gmtime())
    return f"<b>🔥 Quotex Signal 🔥</b>\nAsset: {SYMBOL}\nSignal: {signal}\nPrice: {price}\nTime: {t}\nNote: Educational signal only."

def main():
    last_signal = None
    print("Starting Quotex-style signal bot...")
    while True:
        try:
            df = fetch_price()
            signal, price = compute_signal(df)
            if signal != "HOLD ⚪" and signal != last_signal:
                msg = format_signal_message(signal, price)
                send_message(msg)
                last_signal = signal
                print(f"Sent signal: {signal} at {price}")
            else:
                print(f"{time.strftime('%Y-%m-%d %H:%M:%S')} - No new signal ({signal})")
        except Exception as e:
            print("Error:", e)
        time.sleep(POLL_SECONDS)

if __name__ == "__main__":
    main()