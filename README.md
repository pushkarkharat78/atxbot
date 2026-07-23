import pandas as pd
import numpy as np
import yfinance as yf
import time

class RealTradingBot:
    def __init__(self, ticker="USDBRL=X", initial_balance=100.0, payout_rate=0.85):
        self.ticker = ticker
        self.balance = initial_balance
        self.initial_balance = initial_balance
        self.payout_rate = payout_rate  # Standard binary payout (e.g., 85%)
        self.trade_history = []

    def fetch_market_data(self):
        """Fetches real recent 1-minute data from Yahoo Finance."""
        print(f"Fetching real 1-minute data for {self.ticker}...")
        data = yf.download(self.ticker, period="5d", interval="1m", progress=False)
        
        # Clean multi-index columns if present in newer yfinance versions
        if isinstance(data.columns, pd.MultiIndex):
            data.columns = data.columns.get_level_values(0)
            
        return data.dropna()

    def calculate_indicators(self, df):
        """Calculates Moving Average Crossover and RSI."""
        df['EMA_9'] = df['Close'].ewm(span=9, adjust=False).mean()
        df['EMA_21'] = df['Close'].ewm(span=21, adjust=False).mean()
        
        # Calculate RSI
        delta = df['Close'].diff()
        gain = (delta.where(delta > 0, 0)).rolling(window=14).mean()
        loss = (-delta.where(delta < 0, 0)).rolling(window=14).mean()
        rs = gain / loss
        df['RSI'] = 100 - (100 / (1 + rs))
        
        return df.dropna()

    def generate_signal(self, row, prev_row):
        """Generates a real signal based on EMA crossover and RSI filter."""
        # Bullish: Fast EMA crosses above Slow EMA and RSI is not overbought (< 70)
        if prev_row['EMA_9'] <= prev_row['EMA_21'] and row['EMA_9'] > row['EMA_21'] and row['RSI'] < 70:
            return "CALL"
        # Bearish: Fast EMA crosses below Slow EMA and RSI is not oversold (> 30)
        elif prev_row['EMA_9'] >= prev_row['EMA_21'] and row['EMA_9'] < row['EMA_21'] and row['RSI'] > 30:
            return "PUT"
        return "HOLD"

    def run_backtest_simulation(self):
        """Runs the strategy on historical data to demonstrate real win/loss ratios."""
        df = self.fetch_market_data()
        if df.empty:
            print("Error: Could not retrieve market data.")
            return

        df = self.calculate_indicators(df)
        
        print(f"\n--- Starting Real Simulation ---")
        print(f"Initial Balance: ${self.balance:.2f} | Payout: {self.payout_rate*100}%\n")

        wins = 0
        losses = 0

        # Iterate through candles to simulate trades
        for i in range(1, len(df) - 1):
            prev_row = df.iloc[i-1]
            curr_row = df.iloc[i]
            next_row = df.iloc[i+1] # Used to check if trade won or lost

            signal = self.generate_signal(curr_row, prev_row)

            if signal in ["CALL", "PUT"]:
                stake = self.balance * 0.10  # Risk 10% of current balance (compounding)
                
                # Determine outcome based on real price action of the next candle
                price_went_up = next_row['Close'] > curr_row['Close']
                
                is_win = False
                if signal == "CALL" and price_went_up:
                    is_win = True
                elif signal == "PUT" and not price_went_up:
                    is_win = True

                if is_win:
                    profit = stake * self.payout_rate
                    self.balance += profit
                    wins += 1
                    result = "WIN"
                else:
                    self.balance -= stake
                    losses += 1
                    result = "LOSS"

                self.trade_history.append({
                    "Time": curr_row.name,
                    "Signal": signal,
                    "Stake": round(stake, 2),
                    "Result": result,
                    "New Balance": round(self.balance, 2)
                })

        # Print Summary
        total_trades = wins + losses
        win_rate = (wins / total_trades * 100) if total_trades > 0 else 0
        
        print(f"Total Trades Evaluated: {total_trades}")
        print(f"Wins: {wins} | Losses: {losses}")
        print(f"Real Win Rate: {win_rate:.2f}%")
        print(f"Final Balance: ${self.balance:.2f}")

if __name__ == "__main__":
    bot = RealTradingBot(ticker="USDBRL=X", initial_balance=100.0)
    bot.run_backtest_simulation()
