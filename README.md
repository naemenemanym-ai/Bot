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
