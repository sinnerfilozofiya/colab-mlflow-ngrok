# Colab: MLflow UI + ngrok

Simple guide to open the **MLflow UI** from Google Colab in your browser using **ngrok**.

Colab runs in the cloud, so `http://127.0.0.1:5000` only works *inside* Colab. ngrok gives you a public link you can open on your laptop.

---

## What you need

1. A Google account (for Colab)
2. A free [ngrok](https://ngrok.com) account
3. Your ngrok **auth token** (keep it private — do not commit it to GitHub)

---

## Step 1 — Get an ngrok token

1. Sign up at [https://ngrok.com](https://ngrok.com)
2. Open [Your Authtoken](https://dashboard.ngrok.com/get-started/your-authtoken)
3. Copy the token

Each person needs **their own** token. Do not share it in the repo or group chat.

---

## Step 2 — Open Colab

1. Go to [https://colab.research.google.com](https://colab.research.google.com)
2. Create a new notebook (`File → New notebook`)
3. (Optional) Upload your homework notebook / files with the Colab file sidebar

---

## Step 3 — Start MLflow + ngrok (one cell)

Paste this in a **new cell**, put your token in `NGROK_AUTHTOKEN`, then run it:

```python
# MLflow UI + ngrok in the BACKGROUND (cell finishes immediately)
!pip install -q -U mlflow pyngrok

import os
import subprocess
import time
from IPython.display import display, HTML
from pyngrok import conf, ngrok

# ---- edit ----
NGROK_AUTHTOKEN = "PASTE_YOUR_TOKEN_HERE"
MLFLOW_PORT = 5000
TRACKING_URI = "sqlite:///mlflow.db"
# --------------

os.environ["MLFLOW_TRACKING_URI"] = TRACKING_URI
os.environ["MLFLOW_SERVER_ALLOWED_HOSTS"] = "*"
conf.get_default().auth_token = NGROK_AUTHTOKEN

# clean previous background processes / tunnels
try:
    ngrok.kill()
except Exception:
    pass
subprocess.run(["pkill", "-f", "mlflow ui"], check=False)
subprocess.run(["pkill", "-f", "mlflow.server"], check=False)
time.sleep(1)

# 1) MLflow UI in background
mlflow_log = open("/tmp/mlflow_ui.log", "w")
mlflow_proc = subprocess.Popen(
    [
        "mlflow", "ui",
        "--backend-store-uri", TRACKING_URI,
        "--host", "0.0.0.0",
        "--port", str(MLFLOW_PORT),
        "--allowed-hosts", "*",
    ],
    stdout=mlflow_log,
    stderr=subprocess.STDOUT,
    start_new_session=True,
)
time.sleep(4)

# 2) ngrok tunnel
tunnel = ngrok.connect(addr=str(MLFLOW_PORT), proto="http")
public_url = tunnel.public_url

print("MLflow PID   :", mlflow_proc.pid)
print("MLflow local :", f"http://127.0.0.1:{MLFLOW_PORT}")
print("MLflow public:", public_url)
print("Tracking URI :", TRACKING_URI)
print("Log file     : /tmp/mlflow_ui.log")
display(HTML(f'<a href="{public_url}" target="_blank">{public_url}</a>'))
print("Cell is done — both keep running in the background.")
```

When it finishes you should see a line like:

```text
MLflow public: https://xxxx.ngrok-free.app
```

Click that link (or open it in a new tab).

> First visit on free ngrok may show a short “Visit Site” warning page. Click through once.

---

## Step 4 — Point your notebook code at the same MLflow DB

In **another cell** (your training / homework cells), use the same tracking URI:

```python
import mlflow

mlflow.set_tracking_uri("sqlite:///mlflow.db")
mlflow.set_experiment("hw1-rent-suggestion")  # or any experiment name you like
```

Then log runs as usual (`mlflow.start_run()`, `mlflow.log_metric(...)`, etc.).

Refresh the MLflow public page — your experiment and runs should appear.

---

## Step 5 — Stop everything (when you are done)

```python
from pyngrok import ngrok
import subprocess

ngrok.kill()
subprocess.run(["pkill", "-f", "mlflow ui"], check=False)
print("stopped")
```

Or use Colab: **Runtime → Disconnect and delete runtime**.

---

## Common problems

| Problem | Fix |
|---|---|
| `connection refused` on port 5000 | MLflow did not start. Re-run the setup cell and wait a few seconds. Check `/tmp/mlflow_ui.log`. |
| `Invalid Host header` | Make sure the cell has `--allowed-hosts "*"` and `MLFLOW_SERVER_ALLOWED_HOSTS="*"`. |
| `field schemes not found` | Do **not** use `bind_tls=True`. Use `ngrok.connect(addr=..., proto="http")` as above. |
| Public link opens but old / empty UI | You restarted Colab and got a **new** machine. Re-run the setup cell and use the **new** public URL. Old ngrok links die when the runtime ends. |
| Auth / 403 from ngrok | Token wrong or missing. Copy it again from the ngrok dashboard. |
| Free plan limit | Free ngrok usually allows **one** tunnel. Stop the old one (`ngrok.kill()`) before starting again. |

---

## Security notes (please read)

- Your ngrok URL is **public**. Anyone with the link can open your MLflow UI while the Colab runtime is alive.
- Do **not** put real passwords, private datasets, or API keys in MLflow params/tags you would mind strangers seeing.
- Do **not** commit your `NGROK_AUTHTOKEN` to GitHub.
- When you are finished, stop the tunnel or shut down the Colab runtime.

---

## Short checklist

1. Get ngrok token  
2. New Colab notebook  
3. Paste setup cell → put token → Run  
4. Open **MLflow public** URL  
5. In other cells: `mlflow.set_tracking_uri("sqlite:///mlflow.db")`  
6. Train / log as usual  
7. Stop with the stop cell when done  
