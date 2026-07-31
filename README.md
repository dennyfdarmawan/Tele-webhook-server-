"""
TradingView -> Telegram Webhook Relay
--------------------------------------
Menerima alert JSON dari strategy TradingView (forex_scalping_strategy.pine)
lalu meneruskannya sebagai pesan terformat ke Telegram.

Cara pakai:
  1. pip install flask requests
  2. set env var TELEGRAM_BOT_TOKEN & TELEGRAM_CHAT_ID
  3. python telegram_webhook_server.py
  4. Expose ke internet (ngrok / VPS), pasang URL-nya di Webhook URL alert TradingView
"""

from flask import Flask, request, jsonify
import requests
import os

app = Flask(__name__)

TELEGRAM_BOT_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN", "8962303918:AAGKmad7JWGgFamg1nSCja49Flvx_pWW0qA")
TELEGRAM_CHAT_ID = os.environ.get("TELEGRAM_CHAT_ID", "8188923126")


def send_telegram_message(text: str) -> bool:
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    payload = {"chat_id": TELEGRAM_CHAT_ID, "text": text, "parse_mode": "Markdown"}
    try:
        resp = requests.post(url, json=payload, timeout=10)
        return resp.ok
    except requests.RequestException:
        return False


def format_signal(data: dict) -> str:
    symbol = data.get("symbol", "N/A")
    action = str(data.get("action", "N/A")).upper()
    entry = data.get("entry", "N/A")
    sl = data.get("sl", "N/A")
    tp = data.get("tp", "N/A")
    time_ = data.get("time", "N/A")

    emoji = "🟢" if action == "BUY" else "🔴" if action == "SELL" else "⚪"

    return (
        f"{emoji} *SINYAL SCALPING*\n"
        f"Symbol : `{symbol}`\n"
        f"Action : *{action}*\n"
        f"Entry  : `{entry}`\n"
        f"SL     : `{sl}`\n"
        f"TP     : `{tp}`\n"
        f"Time   : {time_}\n\n"
        f"_Eksekusi manual di MetaTrader sesuai level di atas._"
    )


@app.route("/webhook", methods=["POST"])
def webhook():
    try:
        data = request.get_json(force=True)
    except Exception:
        return jsonify({"status": "error", "message": "invalid JSON"}), 400

    if not data:
        return jsonify({"status": "error", "message": "empty payload"}), 400

    message = format_signal(data)
    sent = send_telegram_message(message)

    if sent:
        return jsonify({"status": "ok"}), 200
    return jsonify({"status": "error", "message": "failed to send telegram message"}), 500


@app.route("/", methods=["GET"])
def health():
    return "Webhook server is running.", 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))

flask
requests
