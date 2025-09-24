# Python
#!/usr/bin/env python3
"""
basic_bot.py
Simplified Binance Futures testnet trading bot (USDT-M).
- Supports MARKET and LIMIT orders.
- Bonus: simple TWAP implementation (splits quantity into slices and places market orders).
- Uses signed REST requests to testnet: https://testnet.binancefuture.com
"""

import os
import sys
import time
import hmac
import json
import hashlib
import logging
import argparse
from logging.handlers import RotatingFileHandler
from urllib.parse import urlencode

import requests

# ----------------------------
# Configuration / constants
# ----------------------------
TESTNET_BASE = "https://testnet.binancefuture.com"
API_ORDER_ENDPOINT = "/fapi/v1/order"
API_QUERY_ORDER = "/fapi/v1/order"

# ----------------------------
# Logging setup
# ----------------------------
LOGFILE = "basic_bot.log"
logger = logging.getLogger("BasicBot")
logger.setLevel(logging.DEBUG)
fmt = logging.Formatter("%(asctime)s | %(levelname)s | %(message)s")

ch = logging.StreamHandler(sys.stdout)
ch.setLevel(logging.INFO)
ch.setFormatter(fmt)
logger.addHandler(ch)

fh = RotatingFileHandler(LOGFILE, maxBytes=2_000_000, backupCount=3)
fh.setLevel(logging.DEBUG)
fh.setFormatter(fmt)
logger.addHandler(fh)


# ----------------------------
# Helper functions
# ----------------------------
def _get_timestamp_ms():
    return int(time.time() * 1000)


def _sign_params(params: dict, secret: str) -> str:
    """
    Create HMAC SHA256 signature for Binance endpoints.
    """
    query = urlencode(params, doseq=True)
    signature = hmac.new(secret.encode("utf-8"), query.encode("utf-8"), hashlib.sha256).hexdigest()
    return signature


def _req(method: str, path: str, api_key: str, api_secret: str, params: dict = None, base=TESTNET_BASE):
    """
    Low-level request wrapper for signed endpoints.
    Logs requests/responses.
    """
    if params is None:
        params = {}
    params["timestamp"] = _get_timestamp_ms()
    query = params.copy()
    signature = _sign_params(query, api_secret)
    query["signature"] = signature
    url = base + path
    headers = {"X-MBX-APIKEY": api_key, "Content-Type": "application/x-www-form-urlencoded"}
    logger.debug("Request -> %s %s -- params: %s", method, url, params)
    try:
        if method.upper() == "POST":
            r = requests.post(url, headers=headers, data=query, timeout=15)
        elif method.upper() == "GET":
            r = requests.get(url, headers=headers, params=query, timeout=15)
        elif method.upper() == "DELETE":
            r = requests.delete(url, headers=headers, params=query, timeout=15)
        else:
            raise ValueError("Unsupported HTTP method: " + method)
    except Exception as e:
        logger.exception("HTTP request failed: %s", e)
        raise

    logger.debug("Response code: %s body: %s", r.status_code, r.text)
    try:
        data = r.json()
    except Exception:
        logger.error("Failed to decode JSON response")
        r.raise_for_status()
    if r.status_code >= 400:
        # Binance often returns JSON error structure: {"code":..., "msg":"..."}
        msg = data.get("msg", str(data))
        logger.error("API error: %s", msg)
        raise RuntimeError(f"API error {r.status_code}: {msg}")
    return data


# ----------------------------
# BasicBot class
# ----------------------------
class BasicBot:
    """
    Basic Binance Futures Testnet bot using REST (signed) endpoints.
    """

    def __init__(self, api_key: str, api_secret: str, testnet: bool = True, base_url: str = None):
        if not api_key or not api_secret:
            raise ValueError("api_key and api_secret are required")
        self.api_key = api_key
        self.api_secret = api_secret
        self.testnet = testnet
        self.base_url = base_url or TESTNET_BASE
        logger.info("BasicBot initialized (testnet=%s) base_url=%s", testnet, self.base_url)

    def place_order(self, symbol: str, side: str, order_type: str, quantity: float, price: float = None,
                    time_in_force: str = "GTC", stop_price: float = None, position_side: str = None):
        """
        Place a new order on futures /fapi/v1/order.

        Args:
            symbol: trading pair like 'BTCUSDT'
            side: 'BUY' or 'SELL'
            order_type: 'MARKET' or 'LIMIT'
            quantity: quantity in contract units (float or str)
            price: required for LIMIT orders
            time_in_force: 'GTC' default
            stop_price: for stop orders (not used in basic MARKET/LIMIT)
            position_side: optional 'LONG'/'SHORT' for hedged accounts

        Returns:
            JSON response from Binance (order info)
        """
        side = side.upper()
        order_type = order_type.upper()

        if side not in ("BUY", "SELL"):
            raise ValueError("side must be BUY or SELL")

        params = {
            "symbol": symbol.upper(),
            "side": side,
            "type": order_type,
            "quantity": str(quantity),
            # timestamp added by low-level wrapper
        }

        if position_side:
            params["positionSide"] = position_side.upper()

        if order_type == "LIMIT":
            if price is None:
                raise ValueError("LIMIT order requires --price")
            params["price"] = str(price)
            params["timeInForce"] = time_in_force
        elif order_type == "MARKET":
            # markets do not need price
            pass
        else:
            raise ValueError("Unsupported order type: " + order_type)

        logger.info("Placing %s %s order for %s qty=%s price=%s", side, order_type, symbol, quantity, price)
        resp = _req("POST", API_ORDER_ENDPOINT, self.api_key, self.api_secret, params, base=self.base_url)
        logger.info("Order placed: %s", resp.get("orderId"))
        return resp

    def query_order(self, symbol: str, order_id: int = None, orig_client_order_id: str = None):
        """
        Query order status.
        """
        params = {"symbol": symbol}
        if order_id:
            params["orderId"] = order_id
        elif orig_client_order_id:
            params["origClientOrderId"] = orig_client_order_id
        else:
            raise ValueError("Either order_id or orig_client_order_id must be provided")
        resp = _req("GET", API_QUERY_ORDER, self.api_key, self.api_secret, params, base=self.base_url)
        return resp

    def twap(self, symbol: str, side: str, total_quantity: float, slices: int = 5, interval: int = 10):
        """
        Simple TWAP: split total_quantity into `slices` equal MARKET orders, spaced by `interval` seconds.
        Returns a list of each order's response.
        """
        if slices < 1:
            raise ValueError("slices must be >= 1")
        per_slice = float(total_quantity) / slices
        results = []
        logger.info("Starting TWAP: %s slices of %s qty each, interval %ss", slices, per_slice, interval)
        for i in range(slices):
            logger.info("TWAP slice %d/%d placing market order qty=%s", i + 1, slices, per_slice)
            try:
                r = self.place_order(symbol=symbol, side=side, order_type="MARKET", quantity=per_slice)
                results.append(r)
            except Exception as e:
                logger.exception("TWAP slice failed: %s", e)
                # continue to next slice or abort depending on policy; here we continue
                results.append({"error": str(e)})
            if i < slices - 1:
                time.sleep(interval)
        logger.info("TWAP completed")
        return results


# ----------------------------
# CLI interface
# ----------------------------
def valid_order_type(x):
    x = x.upper()
    if x not in ("MARKET", "LIMIT", "TWAP"):
        raise argparse.ArgumentTypeError("order type must be MARKET, LIMIT or TWAP")
    return x


def parse_args():
    p = argparse.ArgumentParser(description="Basic Binance Futures Testnet trading bot (Python).")
    p.add_argument("--api-key", help="Binance Testnet API Key (or set BINANCE_API_KEY env var)")
    p.add_argument("--api-secret", help="Binance Testnet API Secret (or set BINANCE_API_SECRET env var)")
    p.add_argument("--base-url", default=TESTNET_BASE, help="Base URL (default: Binance futures testnet)")

    p.add_argument("--symbol", required=True, help="Trading symbol, e.g., BTCUSDT")
    p.add_argument("--side", required=True, choices=["BUY", "SELL"], help="BUY or SELL")
    p.add_argument("--type", dest="order_type", required=True, type=valid_order_type,
                   help="Order type: MARKET, LIMIT, or TWAP (bonus)")

    p.add_argument("--quantity", type=float, required=True, help="Order quantity")
    p.add_argument("--price", type=float, help="Limit price (required for LIMIT orders)")
    p.add_argument("--twap-slices", type=int, default=5, help="TWAP slices (for TWAP)")
    p.add_argument("--twap-interval", type=int, default=10, help="TWAP interval seconds between slices")

    return p.parse_args()


def main():
    args = parse_args()

    api_key = args.api_key or os.environ.get("BINANCE_API_KEY")
    api_secret = args.api_secret or os.environ.get("BINANCE_API_SECRET")
    if not api_key or not api_secret:
        logger.error("API key/secret not provided (env BINANCE_API_KEY / BINANCE_API_SECRET or --api-key/--api-secret)")
        sys.exit(1)

    bot = BasicBot(api_key, api_secret, testnet=True, base_url=args.base_url)

    try:
        if args.order_type == "TWAP":
            res = bot.twap(symbol=args.symbol, side=args.side, total_quantity=args.quantity,
                           slices=args.twap_slices, interval=args.twap_interval)
            print(json.dumps(res, indent=2, default=str))
        else:
            resp = bot.place_order(symbol=args.symbol, side=args.side, order_type=args.order_type,
                                   quantity=args.quantity, price=args.price)
            print(json.dumps(resp, indent=2, default=str))
    except Exception as e:
        logger.exception("Operation failed: %s", e)
        sys.exit(2)


if __name__ == "__main__":
    main()
