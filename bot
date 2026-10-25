import hmac, hashlib, time, os, json, sqlite3
from datetime import datetime
from flask import Flask, request, jsonify, g

app = Flask(__name__)
SECRET = os.environ.get("SEPAY_WEBHOOK_SECRET", "")
BANK_ID = os.environ.get("BANK_ID", "TCB")
ACCOUNT_NO = os.environ.get("ACCOUNT_NO", "0559251796")
DB_PATH = os.environ.get("DB_PATH", "orders.db")

def get_db():
    if "db" not in g:
        g.db = sqlite3.connect(DB_PATH)
        g.db.row_factory = sqlite3.Row
    return g.db

@app.teardown_appcontext
def close_db(e=None):
    db = g.pop("db", None)
    if db: db.close()

def init_db():
    with app.app_context():
        db = get_db()
        db.execute("CREATE TABLE IF NOT EXISTS orders (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id TEXT, order_code TEXT, amount INTEGER, status TEXT DEFAULT 'waiting', qr_url TEXT)")
        db.commit()

def build_qr(amount: int, order_code: str) -> str:
    info = order_code.replace(" ", "%20")
    return f"https://img.vietqr.io/image/{BANK_ID}-{ACCOUNT_NO}-qr_only.svg?amount={amount}&addInfo={info}"

@app.route("/")
def home(): return "Web đang chạy!"

@app.route("/create_order", methods=["POST"])
def create_order():
    data = request.get_json(force=True)
    order_code = f"GDTG {data['name1']} Giao Dich {data['name2']}"
    qr_url = build_qr(int(data['amount']), order_code)
    db = get_db()
    db.execute("INSERT INTO orders (user_id, order_code, amount, qr_url) VALUES (?,?,?,?)", (data['user_id'], order_code, int(data['amount']), qr_url))
    db.commit()
    return jsonify({"order_code": order_code, "qr_url": qr_url})

@app.route("/webhook/sepay", methods=["POST"])
def sepay_webhook():
    sig = request.headers.get("X-SePay-Signature", "")
    ts = int(request.headers.get("X-SePay-Timestamp", 0))
    if abs(time.time() - ts) > 300: return jsonify({"success": False}), 401
    
    body = request.get_data(as_text=True)
    expected = "sha256=" + hmac.new(SECRET.encode(), f"{ts}.{body}".encode(), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, sig): return jsonify({"success": False}), 401

    data = json.loads(body)
    if data.get("transferType") != "credit": return jsonify({"success": True})
    
    amount, content = int(data.get("amount", 0)), data.get("content", "").replace(" ", "")
    db = get_db()
    rows = db.execute("SELECT * FROM orders WHERE status='waiting' AND amount=?", (amount,)).fetchall()
    
    for row in rows:
        if row["order_code"].replace(" ", "") in content:
            db.execute("UPDATE orders SET status='paid' WHERE id=?", (row["id"],))
            db.commit()
            break
    return jsonify({"success": True})

init_db()
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
