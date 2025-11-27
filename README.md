# Bot
Bot bffraz gold
# -*- coding: utf-8 -*-

import pandas as pd
import numpy as np
import requests
import time
from datetime import datetime, timezone

# -------------------
# اندیکاتورهای EMA و RSI

def ema(series, period):
    return series.ewm(span=period, adjust=False).mean()

def rsi(series, period=7):
    delta = series.diff()
    up = np.where(delta > 0, delta, 0.0)
    down = np.where(delta < 0, -delta, 0.0)
    roll_up = pd.Series(up, index=series.index).rolling(period).mean()
    roll_down = pd.Series(down, index=series.index).rolling(period).mean()
    rs = roll_up / (roll_down + 1e-9)
    rsi_val = 100 - (100 / (1 + rs))
    return pd.Series(rsi_val, index=series.index)
import pandas as pd
import numpy as np
import requests
import time
from datetime import datetime, timezone

# ---------------- اندیکاتورها ----------------
def ema(series, period):
    return series.ewm(span=period, adjust=False).mean()

def rsi(series, period=7):
    delta = series.diff()
    up = np.where(delta > 0, delta, 0.0)
    down = np.where(delta < 0, -delta, 0.0)
    roll_up = pd.Series(up).rolling(period).mean()
    roll_down = pd.Series(down).rolling(period).mean()
    rs = roll_up / (roll_down + 1e-9)
    rsi_val = 100 - (100 / (1 + rs))
    return pd.Series(rsi_val, index=series.index)

def bollinger_bands(series, period=20, std=2):
    ma = series.rolling(period).mean()
    sd = series.rolling(period).std(ddof=0)
    upper = ma + std * sd
    lower = ma - std * sd
    return upper, ma, lower

def atr(df, period=14):
    high, low, close = df['High'], df['Low'], df['Close']
    prev_close = close.shift(1)
    tr = pd.concat([
        (high - low),
        (high - prev_close).abs(),
        (low - prev_close).abs()
    ], axis=1).max(axis=1)
    return tr.rolling(period).mean()

# ---------------- منطق سیگنال ----------------
def signal_m1(df_m1: pd.DataFrame,
              atr_min_ratio=0.0003):  # ~0.03% نسبت به قیمت
    df = df_m1.iloc[:-1].copy()  # حذف کندل در حال تشکیل
    close = df['Close']

    ema9 = ema(close, 9)
    ema21 = ema(close, 21)
    rsi7 = rsi(close, 7)
    upper, mid, lower = bollinger_bands(close, 20, 2)
    atr14 = atr(df, 14)

    i = len(df) - 1
    price = close.iloc[i]
    atr_val = atr14.iloc[i]
    atr_ok = atr_val / price >= atr_min_ratio

    ema9_slope_pos = ema9.iloc[i] > ema9.iloc[i-1]
    ema9_slope_neg = ema9.iloc[i] < ema9.iloc[i-1]

    buy_trigger = (ema9.iloc[i] > ema21.iloc[i]) and ema9_slope_pos \
                  and (35 <= rsi7.iloc[i] <= 70) \
                  and (close.iloc[i] > mid.iloc[i]) and atr_ok

    sell_trigger = (ema9.iloc[i] < ema21.iloc[i]) and ema9_slope_neg \
                   and (30 <= rsi7.iloc[i] <= 65) \
                   and (close.iloc[i] < mid.iloc[i]) and atr_ok

    direction = "BUY" if buy_trigger else ("SELL" if sell_trigger else "NONE")

    quality = "C"
    dist_ema = abs(price - ema21.iloc[i]) / price * 100
    slope_strength = abs(ema9.iloc[i] - ema9.iloc[i-1]) / price * 100
    if dist_ema > 0.15 and slope_strength > 0.02:
        quality = "A"
    elif dist_ema > 0.10:
        quality = "B"

    payload = {
        "direction": direction,
        "quality": quality,
        "price": round(price, 2),
        "ema9": round(ema9.iloc[i], 2),
        "ema21": round(ema21.iloc[i], 2),
        "rsi7": round(rsi7.iloc[i], 2),
        "boll_mid": round(mid.iloc[i], 2),
        "atr": round(atr_val, 6),
        "time": df['Time'].iloc[i] if 'Time' in df.columns else datetime.now(timezone.utc).strftime('%Y-%m-%d %H:%M:%S UTC')
    }
    return payload

# ---------------- هشدار تلگرام ----------------
def send_telegram(bot_token: str, chat_id: str, payload: dict):
    if payload.get("direction", "NONE") == "NONE":
        return
    text = (
        f"XAU M1 Signal: {payload['direction']} [{payload['quality']}]\n"
        f"Price: {payload['price']}\n"
        f"EMA9/EMA21: {payload['ema9']}/{payload['ema21']}\n"
        f"RSI7: {payload['rsi7']} | BollMid: {payload['boll_mid']}\n"
        f"ATR: {payload['atr']}\n"
        f"Time: {payload['time']}"
    )
    url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    requests.post(url, json={"chat_id": chat_id, "text": text})

# ---------------- حلقه اصلی ----------------
def run_loop_m1(fetch_m1, bot_token, chat_id, sleep_sec=20):
    last_dir = None
    last_time = None
    while True:
        try:
            df_m1 = fetch_m1()  # باید داده‌های M1 (OHLC) برگرداند
            sig = signal_m1(df_m1)
            cur_dir = sig.get("direction", "NONE")
            cur_time = sig.get("time")

            if cur_dir != "NONE":
                if cur_dir != last_dir or cur_time != last_time:
                    send_telegram(bot_token, chat_id, sig)
                    last_dir, last_time = cur_dir, cur_time
        except Exception as e:
            print("Error:", e)
        time.sleep(sleep_sec)
