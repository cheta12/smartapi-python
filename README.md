# file: nifty_call_green_one_trade.py
import os, json, math, time, logging
import datetime as dt
from smartapi import SmartConnect

# ================== USER CONFIG (EDIT) ==================
API_KEY     = "t9uXoXR6"
CLIENT_ID   = "C32832"
PASSWORD    = "9689"
TOTP_SECRET = "YOUR_TOTP_SECRET"  # 2FA (Google Authenticator). If not used, pass "".

UNDERLYING        = "NIFTY"       # Only NIFTY handled here
EXCHANGE_EQ       = "NSE"
EXCHANGE_OI       = "NFO"
INTERVAL          = "FIVE_MINUTE" # Candle timeframe
ENTRY_EARLIEST    = dt.time(9, 30)
ENTRY_LATEST      = dt.time(11, 30)
EXIT_TIME         = dt.time(15, 10)

TARGET_PCT        = 0.10          # +10% on option premium
CAPITAL_FRACTION  = 0.20          # 20% of available cash (1.0 = full capital; risky!)
NIFTY_LOT_SIZE    = 50            # Verify with contract master

# One-trade-per-day state
STATE_FILE            = "nifty_one_trade_state.json"
MAX_TRADES_PER_DAY    = 1

# Angel One known tokens (verify)
NIFTY_SPOT_TOKEN  = "26000"

# ================== STATE HELPERS ==================
def _load_state():
    if os.path.exists(STATE_FILE):
        try:
            return json.load(open(STATE_FILE, "r"))
        except Exception:
            return {}
    return {}

def _save_state(d):
    with open(STATE_FILE, "w") as f:
        json.dump(d, f)

def already_traded_today():
    s = _load_state()
    today = dt.date.today().isoformat()
    return s.get("date") == today and s.get("trades", 0) >= MAX_TRADES_PER_DAY

def mark_trade_done(order_ids=None):
    s = _load_state()
    s["date"] = dt.date.today().isoformat()
    s["trades"] = MAX_TRADES_PER_DAY
    s["orders"] = order_ids or []
    _save_state(s)

# ================== SMARTAPI SESSION ==================
def login():
    smart = SmartConnect(api_key=API_KEY)
    smart.generateSession(CLIENT_ID, PASSWORD, TOTP_SECRET)
    logging.info("SmartAPI login OK")
    return smart

# ================== BROKER HELPERS ==================
def get_funds_available(smart):
    try:
        rms = smart.rmsLimit()
        # Some accounts use 'availablecash' or 'net' etc.
        cash = float(rms["data"].get("availablecash") or rms["data"].get("net") or 0.0)
        return max(0.0, cash)
    except Exception as e:
        logging.warning(f"Funds fetch failed: {e}")
        return 0.0

def get_spot_ltp(smart):
    r = smart.ltpData(EXCHANGE_EQ, UNDERLYING, NIFTY_SPOT_TOKEN)
    return float(r["data"]["ltp"])

def get_candles(smart, token, start_dt, end_dt, interval=INTERVAL):
    params = {
        "exchange": EXCHANGE_EQ,
        "symboltoken": token,
        "interval": interval,
        "fromdate": start_dt.strftime("%Y-%m-%d %H:%M"),
        "todate":   end_dt.strftime("%Y-%m-%d %H:%M"),
    }
    data = smart.getCandleData(params)
    return data["data"]  # [time, open, high, low, close, volume]

def first_green_after_930(smart):
    today = dt.datetime.now()
    start = dt.datetime(today.year, today.month, today.day, 9, 15)
    end   = dt.datetime(today.year, today.month, today.day, 11, 30)
    candles = get_candles(smart, NIFTY_SPOT_TOKEN, start, end)

    for t, o, h, l, c, v in candles:
        ts = t.replace("T", " ").split("+")[0]
        ct = dt.datetime.strptime(ts, "%Y-%m-%d %H:%M:%S")
        if ct.time() <= ENTRY_EARLIEST:
            continue
        if c > o:  # green candle
            return {"time": ct, "open": float(o), "high": float(h), "low": float(l), "close": float(c)}
    return None

def weekly_expiry(today=None):
    """Next/this Thursday expiry (weekly index options)."""
    today = today or dt.datetime.now()
    thursday = today + dt.timedelta(days=(3 - today.weekday()))
    if thursday.date() < today.date():
        thursday += dt.timedelta(days=7)
    return dt.datetime(thursday.year, thursday.month, thursday.day)

def atm_strike(spot):
    # NIFTY: 50-point strikes
    return int(round(spot / 50.0) * 50)

def option_symbol(strike, expiry_dt):
    # Typical format. Cross-check with contract master on your account.
    # Example: NIFTY25AUG24500CE
    return f"NIFTY{expiry_dt:%y%b}{strike}CE".upper()

def search_token(smart, tradingsymbol):
    res = smart.searchScrip(EXCHANGE_OI, tradingsymbol)
    for x in res.get("data", []):
        if x.get("tradingsymbol") == tradingsymbol:
            return x["symboltoken"]
    raise ValueError(f"Token not found for {tradingsymbol}")

def option_ltp(smart, tsym, token):
    r = smart.ltpData(EXCHANGE_OI, tsym, token)
    return float(r["data"]["ltp"])

def place_market(smart, tsym, token, side, qty):
    """side = 'BUY' or 'SELL'"""
    return smart.placeOrder(
        variety="NORMAL",
        tradingsymbol=tsym,
        symboltoken=token,
        transactiontype=side,
        exchange=EXCHANGE_OI,
        ordertype="MARKET",
        producttype="INTRADAY",
        duration="DAY",
        quantity=int(qty)
    )

def place_slm_exit(smart, tsym, token, side, qty, trigger):
    """
    Place Stop-Loss-Market exit. For a long CALL, side should be SELL.
    Some setups require STOPLOSS_LIMIT; adjust if broker rejects SLM.
    """
    return smart.placeOrder(
        variety="STOPLOSS",
        tradingsymbol=tsym,
        symboltoken=token,
        transactiontype=side,
        exchange=EXCHANGE_OI,
        producttype="INTRADAY",
        duration="DAY",
        ordertype="STOPLOSS_MARKET",
        quantity=int(qty),
        triggerprice=round(float(trigger), 1)
    )

# ================== POSITION SIZING ==================
def compute_qty(smart, option_price):
    funds = get_funds_available(smart)
    use_cap = funds * CAPITAL_FRACTION
    lots = math.floor(use_cap / (option_price * NIFTY_LOT_SIZE))
    qty = max(0, lots) * NIFTY_LOT_SIZE
    return qty, funds, use_cap

# ================== MAIN STRATEGY ==================
def run_day():
    if already_traded_today():
        logging.info("Already traded today (one-trend rule). Exiting.")
        return

    smart = login()

    # Wait until >= 9:30 if script started earlier
    while dt.datetime.now().time() < ENTRY_EARLIEST:
        logging.info("Waiting for 9:30 AM...")
        time.sleep(5)

    # Identify first green candle after 9:30 (till 11:30)
    sig = first_green_after_930(smart)
    if not sig:
        logging.info("No green candle till 11:30 → No trade.")
        return
    if sig["time"].time() > ENTRY_LATEST:
        logging.info("Green candle after 11:30 → Skipping by rule.")
        return

    # Build option contract (ATM CE of current week)
    spot   = get_spot_ltp(smart)
    strike = atm_strike(spot)
    expiry = weekly_expiry()
    ce_sym = option_symbol(strike, expiry)
    ce_tok = search_token(smart, ce_sym)

    entry_price = option_ltp(smart, ce_sym, ce_tok)
    qty, funds, use_cap = compute_qty(smart, entry_price)
    if qty <= 0:
        logging.warning(f"Insufficient funds. Funds={funds:.2f}, use_cap={use_cap:.2f}, price≈{entry_price:.2f}")
        return

    logging.info(f"Signal candle: {sig}")
    logging.info(f"Spot≈{spot:.1f} | ATM {strike} | {ce_sym} LTP≈{entry_price:.2f} | Qty={qty}")

    # ---- Option-price SL derived from green-candle low (approx) ----
    # Underlying candle range:
    underline_down = max(0.0, sig["close"] - sig["low"])
    # Map underlying % drop to option SL % (approx). Cap 4%–12% to avoid extreme values.
    approx_pct = min(0.12, max(0.04, underline_down / max(1.0, sig["close"])))
    sl_price   = entry_price * (1 - approx_pct)
    target_px  = entry_price * (1 + TARGET_PCT)

    # Place BUY (market)
    buy_resp = place_market(smart, ce_sym, ce_tok, "BUY", qty)
    logging.info(f"BUY placed: {buy_resp}")

    # Place broker-side SL-M exit
    try:
        sl_resp = place_slm_exit(smart, ce_sym, ce_tok, "SELL", qty, trigger=sl_price)
        logging.info(f"SL-M placed at ≈{sl_price:.2f}: {sl_resp}")
    except Exception as e:
        logging.warning(f"SL-M placement failed: {e} (will monitor & exit manually)")

    # Monitor for target / time exit
    order_ids = [str(buy_resp)]
    while True:
        now = dt.datetime.now().time()
        if now >= EXIT_TIME:
            logging.info("Time exit (3:10 PM). Squaring off.")
            try:
                place_market(smart, ce_sym, ce_tok, "SELL", qty)
            except Exception as e:
                logging.error(f"Time exit SELL failed: {e}")
            mark_trade_done(order_ids=order_ids)
            break

        try:
            cur = option_ltp(smart, ce_sym, ce_tok)
        except Exception as e:
            logging.warning(f"LTP fetch failed: {e}")
            time.sleep(2)
            continue

        if cur >= target_px:
            logging.info(f"Target hit: {cur:.2f} ≥ {target_px:.2f}. Booking profit.")
            try:
                place_market(smart, ce_sym, ce_tok, "SELL", qty)
                # Ideally cancel SL order here via cancel API using returned ID.
            except Exception as e:
                logging.error(f"Target SELL failed: {e}")
            mark_trade_done(order_ids=order_ids)
            break

        # If broker-side SL triggers, position may already be closed;
        # for robustness you'd poll positions and break if FLAT.
        time.sleep(2)

# ================== ENTRY POINT ==================
if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(message)s")
    run_day() 
