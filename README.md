import os
import time
from SmartApi import SmartConnect
import pyotp

# API Credentials & Setup
API_KEY = os.getenv("API_KEY", "YOUR_API_KEY")
CLIENT_CODE = os.getenv("CLIENT_CODE", "YOUR_CLIENT_ID")
PASSWORD = os.getenv("PASSWORD", "YOUR_PASSWORD")
TOTP_SECRET = os.getenv("TOTP_SECRET", "YOUR_TOTP_SECRET_KEY")

smart_api = SmartConnect(api_key=API_KEY)

def login():
    try:
        totp = pyotp.TOTP(TOTP_SECRET).now()
        data = smart_api.generateSession(CLIENT_CODE, PASSWORD, totp)
        if data and data.get('status'):
            print("Login successful!")
            return True
        else:
            print("Login failed:", data)
            return False
    except Exception as e:
        print(f"Error during login: {e}")
        return False

def get_ltp(exchange, tradingsymbol, symboltoken):
    try:
        data = smart_api.ltpData(exchange, tradingsymbol, symboltoken)
        if data and data.get('status'):
            return float(data['data']['ltp'])
        else:
            print(f"Failed to fetch LTP: {data}")
            return None
    except Exception as e:
        print(f"Exception fetching LTP: {e}")
        return None

def exit_position(exchange, tradingsymbol, symboltoken, quantity, transaction_type="SELL"):
    try:
        order_params = {
            "variety": "NORMAL",
            "tradingsymbol": tradingsymbol,
            "symboltoken": symboltoken,
            "transactiontype": transaction_type,
            "exchange": exchange,
            "ordertype": "MARKET",
            "producttype": "INTRADAY",
            "duration": "DAY",
            "price": "0",
            "quantity": str(quantity)
        }
        order_id = smart_api.placeOrder(order_params)
        print(f"Exit Order Placed Successfully! Order ID: {order_id}")
        return order_id
    except Exception as e:
        print(f"Error placing exit order: {e}")
        return None

def monitor_trade(exchange, tradingsymbol, symboltoken, entry_price, target_pts, sl_pts, quantity):
    target_price = entry_price + target_pts
    sl_price = entry_price - sl_pts

    print(f"\n--- Monitoring Started for {tradingsymbol} ---")
    print(f"Entry: {entry_price} | Target Price: {target_price} | Stop Loss Price: {sl_price}\n")

    while True:
        ltp = get_ltp(exchange, tradingsymbol, symboltoken)
        
        if ltp is not None:
            print(f"Current LTP: {ltp}")

            if ltp >= target_price:
                print("Target Hit! Position Exit...")
                exit_position(exchange, tradingsymbol, symboltoken, quantity, transaction_type="SELL")
                break

            elif ltp <= sl_price:
                print("Stop Loss Hit! Position Exit...")
                exit_position(exchange, tradingsymbol, symboltoken, quantity, transaction_type="SELL")
                break

        time.sleep(1)

if __name__ == "__main__":
    if login():
        EXCHANGE = "NFO"
        TRADING_SYMBOL = "NIFTY20AUG2624500CE"
        SYMBOL_TOKEN = "12345"
        
        ENTRY_PRICE = 100.0
        TARGET_POINTS = 20.0
        SL_POINTS = 10.0
        QTY = 25

        monitor_trade(EXCHANGE, TRADING_SYMBOL, SYMBOL_TOKEN, ENTRY_PRICE, TARGET_POINTS, SL_POINTS, QTY)
        
