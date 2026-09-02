"""
App di segnali di trading - Mercati12222
-----------------------------------------
Strategia: incrocio medie mobili (SMA 20 / SMA 50) con filtro RSI(14)
Dati di mercato: Twelve Data
Invio avvisi: Resend (email)

Come funziona:
1. Un servizio esterno gratuito (es. cron-job.org) chiama l'indirizzo
   https://TUO-APP.onrender.com/check-signals ogni tot minuti.
2. L'app scarica gli ultimi prezzi per ogni simbolo in SYMBOLS.
3. Calcola SMA20, SMA50 e RSI14.
4. Se c'è un nuovo incrocio (golden cross / death cross) confermato dal
   filtro RSI, invia una email con il segnale.
5. Salva l'ultimo segnale in un file (signals_state.json) per non
   mandare la stessa email più volte di fila.
"""

import os
import json
import requests
import pandas as pd
from flask import Flask, jsonify

app = Flask(__name__)

# ---------------------------------------------------------------------
# CONFIGURAZIONE - modifica qui i simboli che vuoi seguire
# ---------------------------------------------------------------------
SYMBOLS = [
    "XAU/USD",   # oro
    "WTI/USD",   # petrolio
    "EUR/USD",   # forex
]

TWELVE_DATA_API_KEY = os.environ.get("TWELVE_DATA_API_KEY")
RESEND_API_KEY = os.environ.get("RESEND_API_KEY")
ALERT_EMAIL_TO = os.environ.get("ALERT_EMAIL_TO")
ALERT_EMAIL_FROM = os.environ.get("ALERT_EMAIL_FROM")

STATE_FILE = "signals_state.json"


# ---------------------------------------------------------------------
# DATI DI MERCATO
# ---------------------------------------------------------------------
def get_price_data(symbol, interval="1h", output_size=100):
    """Scarica le candele storiche da Twelve Data per un simbolo."""
    url = "https://api.twelvedata.com/time_series"
    params = {
        "symbol": symbol,
        "interval": interval,
        "outputsize": output_size,
        "apikey": TWELVE_DATA_API_KEY,
    }
    resp = requests.get(url, params=params, timeout=20)
    data = resp.json()

    if "values" not in data:
        raise ValueError(f"Errore dati per {symbol}: {data}")

    df = pd.DataFrame(data["values"])
    df["datetime"] = pd.to_datetime(df["datetime"])
    df["close"] = df["close"].astype(float)
    df = df.sort_values("datetime").reset_index(drop=True)
    return df


# ---------------------------------------------------------------------
# INDICATORI
# ---------------------------------------------------------------------
def add_indicators(df):
    df["sma20"] = df["close"].rolling(window=20).mean()
    df["sma50"] = df["close"].rolling(window=50).mean()

    delta = df["close"].diff()
    gain = delta.clip(lower=0)
    loss = -delta.clip(upper=0)
    avg_gain = gain.rolling(window=14).mean()
    avg_loss = loss.rolling(window=14).mean()
    rs = avg_gain / avg_loss
    df["rsi14"] = 100 - (100 / (1 + rs))

    return df


def get_signal(df):
    """Ritorna 'BUY', 'SELL' o None guardando l'ultimo incrocio + RSI."""
    if len(df) < 51:
        return None  # non abbastanza dati per SMA50

    last = df.iloc[-1]
    prev = df.iloc[-2]

    crossed_up = prev["sma20"] <= prev["sma50"] and last["sma20"] > last["sma50"]
    crossed_down = prev["sma20"] >= prev["sma50"] and last["sma20"] < last["sma50"]

    if crossed_up and last["rsi14"] < 70:
        return "BUY"
    if crossed_down and last["rsi14"] > 30:
        return "SELL"
    return None


# ---------------------------------------------------------------------
# STATO (per non inviare email duplicate)
# ---------------------------------------------------------------------
def load_state():
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE, "r") as f:
            return json.load(f)
    return {}


def save_state(state):
    with open(STATE_FILE, "w") as f:
        json.dump(state, f)


# ---------------------------------------------------------------------
# EMAIL (Resend)
# ---------------------------------------------------------------------
def send_email_alert(symbol, signal, price):
    url = "https://api.resend.com/emails"
    headers = {
        "Authorization": f"Bearer {RESEND_API_KEY}",
        "Content-Type": "application/json",
    }
    payload = {
        "from": ALERT_EMAIL_FROM,
        "to": [ALERT_EMAIL_TO],
        "subject": f"Segnale {signal} su {symbol}",
        "html": (
            f"<h2>Nuovo segnale: {signal}</h2>"
            f"<p>Simbolo: <b>{symbol}</b></p>"
            f"<p>Prezzo attuale: {price}</p>"
            f"<p>Strategia: incrocio SMA20/SMA50 + filtro RSI14</p>"
        ),
    }
    resp = requests.post(url, headers=headers, json=payload, timeout=20)
    return resp.status_code, resp.text


# ---------------------------------------------------------------------
# ROTTA PRINCIPALE
# ---------------------------------------------------------------------
@app.route("/check-signals")
def check_signals():
    state = load_state()
    results = {}

    for symbol in SYMBOLS:
        try:
            df = get_price_data(symbol)
            df = add_indicators(df)
            signal = get_signal(df)
            last_price = df.iloc[-1]["close"]

            if signal and state.get(symbol) != signal:
                send_email_alert(symbol, signal, last_price)
                state[symbol] = signal
                results[symbol] = f"Nuovo segnale inviato: {signal}"
            else:
                results[symbol] = "Nessun nuovo segnale"

        except Exception as e:
            results[symbol] = f"Errore: {str(e)}"

    save_state(state)
    return jsonify(results)


@app.route("/test-email")
def test_email():
    """
    ROTTA TEMPORANEA - solo per test.
    Manda subito una email finta per verificare che Resend funzioni.
    Una volta verificato che l'email arriva, puoi togliere questa
    funzione e la riga @app.route sopra.
    """
    status_code, response_text = send_email_alert("TEST", "BUY", 1234.56)
    return jsonify({
        "status_code": status_code,
        "risposta_resend": response_text,
        "nota": "Se status_code è 200, l'email è stata inviata. Controlla la casella."
    })


@app.route("/")
def home():
    return "App segnali di trading attiva. Usa /check-signals per eseguire il controllo."


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))