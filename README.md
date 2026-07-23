import time
import requests
import pandas as pd
import ta  # Technical analysis library

# ==========================================
# ⚙️ ATX BOT CONFIGURATION
# ==========================================
BOT_NAME = "ATX Bot"
TELEGRAM_BOT_TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"  # Replace with token from @BotFather
TELEGRAM_CHAT_ID = "YOUR_TELEGRAM_CHAT_ID"      # Channel or Group ID (e.g., -100xxxxxxxxxx)

# Locked Target Pair & Timeframe Configuration
TARGET_PAIR = "USDBRL OTC"
DEFAULT_EXPIRY = "1 Minute"
DEFAULT_TIMEFRAME = "M1"


# ==========================================
# 📊 TECHNICAL ANALYSIS & SIGNAL LOGIC
# ==========================================
def analyze_market(ohlc_df: pd.DataFrame) -> dict:
    """
    Analyzes 1-minute OHLC market data for USDBRL OTC using RSI and EMA Crossover.
    ohlc_df must contain columns: ['open', 'high', 'low', 'close']
    """
    # Calculate Indicators
    ohlc_df['rsi'] = ta.momentum.rsi(ohlc_df['close'], window=14)
    ohlc_df['ema_fast'] = ta.trend.ema_indicator(ohlc_df['close'], window=9)
    ohlc_df['ema_slow'] = ta.trend.ema_indicator(ohlc_df['close'], window=21)

    latest = ohlc_df.iloc[-1]
    previous = ohlc_df.iloc[-2]

    rsi_val = round(latest['rsi'], 2)
    signal = "WAIT"

    # CALL (BUY) Condition: Oversold RSI + Bullish EMA Crossover
    if latest['rsi'] < 35 and previous['ema_fast'] <= previous['ema_slow'] and latest['ema_fast'] > latest['ema_slow']:
        signal = "CALL 🟢"

    # PUT (SELL) Condition: Overbought RSI + Bearish EMA Crossover
    elif latest['rsi'] > 65 and previous['ema_fast'] >= previous['ema_slow'] and latest['ema_fast'] < latest['ema_slow']:
        signal = "PUT 🔴"

    return {
        "signal": signal,
        "rsi": rsi_val,
        "price": latest['close']
    }


# ==========================================
# 📨 TELEGRAM MESSAGE DISPATCHER
# ==========================================
def send_atx_signal(signal_data: dict):
    """Formats and sends the signal message to Telegram."""
    
    if signal_data["signal"] == "WAIT":
        print("[ATX Bot] Market condition neutral. Waiting for next candle...")
        return

    message = (
        f"🤖 *{BOT_NAME} SIGNAL* 🤖\n"
        f"━━━━━━━━━━━━━━━━━━━\n"
        f"🔤 *Asset:* {TARGET_PAIR}\n"
        f"📊 *Signal:* *{signal_data['signal']}*\n"
        f"⏱️ *Expiry Time:* {DEFAULT_EXPIRY}\n"
        f"📈 *Timeframe:* {DEFAULT_TIMEFRAME}\n"
        f"📉 *Current RSI:* {signal_data['rsi']}\n"
        f"━━━━━━━━━━━━━━━━━━━\n"
        f"⚠️ Execute entry immediately at the start of the next candle."
    )

    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    payload = {
        "chat_id": TELEGRAM_CHAT_ID,
        "text": message,
        "parse_mode": "Markdown"
    }

    try:
        response = requests.post(url, json=payload)
        if response.status_code == 200:
            print(f"[ATX Bot] Signal sent successfully: {signal_data['signal']}")
        else:
            print(f"[ATX Bot] Telegram Error: {response.text}")
    except Exception as e:
        print(f"[ATX Bot] Network Exception: {e}")


# ==========================================
# 🚀 MAIN LOOP ENGINE
# ==========================================
def run_atx_bot():
    print(f"🚀 {BOT_NAME} Started...")
    print(f"🎯 Lock Target: {TARGET_PAIR} | Expiry: {DEFAULT_EXPIRY}")

    while True:
        try:
            # Note: Fetch real-time USDBRL OTC candle data from your data feed/API
            # Below is mock data structural format for representation
            sample_ohlc_data = pd.DataFrame({
                'open': [5.120, 5.121, 5.118, 5.115, 5.112],
                'high': [5.122, 5.123, 5.119, 5.116, 5.114],
                'low': [5.119, 5.117, 5.114, 5.111, 5.109],
                'close': [5.121, 5.118, 5.115, 5.112, 5.110]
            })

            # Process indicators and generate signal
            analysis = analyze_market(sample_ohlc_data)
            
            # Send alert if actionable signal was found
            send_atx_signal(analysis)

        except Exception as err:
            print(f"[ATX Bot Error]: {err}")

        # Wait 60 seconds for the next 1-minute candle
        time.sleep(60)


if _name_ == "_main_":
    run_atx_bot()
