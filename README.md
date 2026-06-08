"""
app.py — ThutoClub LS  Flask Backend (Supabase edition)
========================================================
Routes implemented
------------------
  GET  /                  → Sign-in / sign-up page  (index.html)
  POST /login             → JSON { email, password }
  POST /signup            → JSON { first_name, last_name, email, password }
  POST /forgot-password   → JSON { email }
  GET  /reset-password    → token link from e-mail
  POST /reset-password    → JSON { token, password }
  GET  /logout            → clears session
  GET  /dashboard         → @login_required
  GET  /subjects          → public listing stub
  GET  /years             → public listing stub
  GET  /about             → public page stub
  GET  /terms             → public page stub
  GET  /policy            → public page stub
  GET  /auth/google       → stub (Google OAuth placeholder)

Quick-start
-----------
  pip install flask flask-bcrypt itsdangerous flask-mail supabase
  python app.py
  # visit http://localhost:5000
"""

import hashlib
import os
from datetime import datetime, timedelta
from functools import wraps

from flask import (
    Flask, jsonify, redirect, render_template,
    render_template_string, request, session, url_for
)
from flask_bcrypt import Bcrypt
from itsdangerous import URLSafeTimedSerializer, BadSignature, SignatureExpired
from supabase import create_client, Client

# ──────────────────────────────────────────────────────────────────────
# App setup
# ──────────────────────────────────────────────────────────────────────
app = Flask(__name__)

app.secret_key = os.environ.get("SECRET_KEY", "0cf99f4bb8125ec86287139bfe34a931")
app.config["SESSION_COOKIE_HTTPONLY"] = True
app.config["SESSION_COOKIE_SAMESITE"] = "Lax"

# ──────────────────────────────────────────────────────────────────────
# Supabase setup
# ──────────────────────────────────────────────────────────────────────
# 👇 Paste your Supabase Project URL here  (looks like https://xxxxxxxxxxxx.supabase.co)
SUPABASE_URL = os.environ.get("SUPABASE_URL", "YOUR_SUPABASE_URL_HERE")

# 👇 Paste your Supabase anon/public API key here
SUPABASE_KEY = os.environ.get("SUPABASE_KEY", "YOUR_SUPABASE_ANON_KEY_HERE")

supabase: Client = create_client(SUPABASE_URL, SUPABASE_KEY)

# ── Bcrypt (password hashing) ──────────────────────────────────────────
bcrypt = Bcrypt(app)

# ── Token serialiser (password-reset links) ───────────────────────────
ts = URLSafeTimedSerializer(app.secret_key)
RESET_TOKEN_MAX_AGE = 3600  # 1 hour in seconds

# ──────────────────────────────────────────────────────────────────────
# Optional: Flask-Mail  (e-mails)
# ──────────────────────────────────────────────────────────────────────
app.config.update(
    MAIL_SERVER         = "smtp.gmail.com",
    MAIL_PORT           = 587,
    MAIL_USE_TLS        = True,
    MAIL_USERNAME       = os.environ.get("MAIL_USERNAME"),
    MAIL_PASSWORD       = os.environ.get("MAIL_PASSWORD"),
    MAIL_DEFAULT_SENDER = ("ThutoClub LS", os.environ.get("MAIL_USERNAME")),
)
# from flask_mail import Mail, Message
# mail = Mail(app)

# ──────────────────────────────────────────────────────────────────────
# Database helpers — now talking to Supabase
# ──────────────────────────────────────────────────────────────────────

def get_user_by_email(email: str):
    res = supabase.table("users").select("*").eq("email", email.strip().lower()).execute()
    return res.data[0] if res.data else None


def get_user_by_id(user_id):
    res = supabase.table("users").select("*").eq("id", user_id).execute()
    return res.data[0] if res.data else None


# ──────────────────────────────────────────────────────────────────────
# Auth decorator
# ──────────────────────────────────────────────────────────────────────

def login_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if "user_id" not in session:
            return redirect(url_for("index"))
        return f(*args, **kwargs)
    return decorated


# ──────────────────────────────────────────────────────────────────────
# Helper: send password-reset e-mail
# ──────────────────────────────────────────────────────────────────────

def send_reset_email(email: str, reset_url: str):
    """
    Replace the print() below with Flask-Mail code once configured:

        msg = Message("Reset your ThutoClub LS password", recipients=[email])
        msg.body = (
            "Hi,\\n\\nClick the link below to reset your password.\\n"
            "It expires in 1 hour.\\n\\n" + reset_url +
            "\\n\\nIf you didn't request this, ignore this message."
        )
        mail.send(msg)
    """
    print(f"\n[DEV] Password-reset link for {email}:\n  {reset_url}\n")


# ──────────────────────────────────────────────────────────────────────
# Routes — public pages
# ──────────────────────────────────────────────────────────────────────

@app.route("/")
def index():
    if "user_id" in session:
        return redirect(url_for("dashboard"))  # ✅ fixed: was "dashboard.html"
    try:
        return render_template("index.html")
    except Exception:
        html_path = os.path.join(os.path.dirname(__file__), "index.html")
        if os.path.exists(html_path):
            with open(html_path, encoding="utf-8") as f:
                return f.read()
        return "index.html not found — place it in the same directory as app.py", 404


@app.route("/subjects")
def subjects():
    return render_template("subjects.html")


@app.route("/years")
def years():
    return render_template("years.html")


@app.route("/about")
def about():
    return render_template("about.html")


@app.route("/terms")
def terms():
    return render_template("terms.html")


@app.route("/policy")
def policy():
    return render_template("policy.html")


@app.route("/mathematics")
def mathematics():
    return render_template("mathematics.html")


@app.route("/english-language")
def english_language():
    return render_template("english-language.html")


# ── University routes (no more .html extensions) ──────────────────────

@app.route("/nul")
def nul():
    return render_template("nul.html")


@app.route("/botho")
def botho():
    return render_template("botho.html")


@app.route("/limkokwing")
def limkokwing():
    return render_template("limkokwing.html")


@app.route("/other-institutions")
def other_institutions():
    return render_template("other-institutions.html")


# ──────────────────────────────────────────────────────────────────────
# Routes — auth
# ──────────────────────────────────────────────────────────────────────

@app.route("/login", methods=["POST"])
def login():
    """
    POST /login
    Body: { "email": "...", "password": "..." }
    Returns:
        200  { "ok": true }           → sets session, client redirects to /dashboard
        401  { "ok": false, "error": "..." }
    """
    data     = request.get_json(silent=True) or {}
    email    = (data.get("email") or "").strip().lower()
    password = data.get("password") or ""

    if not email or not password:
        return jsonify({"ok": False, "error": "Email and password are required."}), 401

    user = get_user_by_email(email)
    if not user or not bcrypt.check_password_hash(user["password_hash"], password):
        return jsonify({"ok": False, "error": "Invalid email or password."}), 401

    session.clear()
    session["user_id"]    = user["id"]
    session["user_email"] = user["email"]
    session["user_name"]  = user["first_name"]
    session.permanent     = True
    app.permanent_session_lifetime = timedelta(days=30)

    return jsonify({"ok": True})


@app.route("/signup", methods=["POST"])
def signup():
    """
    POST /signup
    Body: { "first_name": "...", "last_name": "...", "email": "...", "password": "..." }
    Returns:
        200  { "ok": true }           → sets session, client redirects to /dashboard
        409  { "ok": false, "error": "..." }
    """
    data       = request.get_json(silent=True) or {}
    first_name = (data.get("first_name") or "").strip()
    last_name  = (data.get("last_name")  or "").strip()
    email      = (data.get("email")      or "").strip().lower()
    password   = data.get("password")   or ""

    if not first_name or not last_name:
        return jsonify({"ok": False, "error": "First and last name are required."}), 409
    if not email or "@" not in email:
        return jsonify({"ok": False, "error": "A valid email address is required."}), 409
    if len(password) < 8:
        return jsonify({"ok": False, "error": "Password must be at least 8 characters."}), 409

    if get_user_by_email(email):
        return jsonify({"ok": False, "error": "An account with that email already exists."}), 409

    pw_hash = bcrypt.generate_password_hash(password).decode("utf-8")

    res = supabase.table("users").insert({
        "first_name":    first_name,
        "last_name":     last_name,
        "email":         email,
        "password_hash": pw_hash,
        "downloads":     1,
        "created_at":    datetime.utcnow().isoformat(),
    }).execute()

    if not res.data:
        return jsonify({"ok": False, "error": "Could not create account. Please try again."}), 500

    user = res.data[0]

    session.clear()
    session["user_id"]    = user["id"]
    session["user_email"] = user["email"]
    session["user_name"]  = user["first_name"]
    session.permanent     = True
    app.permanent_session_lifetime = timedelta(days=30)

    return jsonify({"ok": True})


@app.route("/forgot-password", methods=["POST"])
def forgot_password():
    """
    POST /forgot-password
    Body: { "email": "..." }
    Always returns 200 { "ok": true }  (never reveal if e-mail exists)
    """
    data  = request.get_json(silent=True) or {}
    email = (data.get("email") or "").strip().lower()

    user = get_user_by_email(email)
    if user:
        token      = ts.dumps(email, salt="password-reset")
        reset_url  = url_for("reset_password", token=token, _external=True)
        token_hash = hashlib.sha256(token.encode()).hexdigest()
        expires_at = (datetime.utcnow() + timedelta(seconds=RESET_TOKEN_MAX_AGE)).isoformat()

        # Invalidate any previous unused tokens for this user
        supabase.table("reset_tokens") \
            .update({"used": True}) \
            .eq("user_id", user["id"]) \
            .eq("used", False) \
            .execute()

        # Insert new token
        supabase.table("reset_tokens").insert({
            "user_id":    user["id"],
            "token_hash": token_hash,
            "expires_at": expires_at,
            "used":       False,
        }).execute()

        send_reset_email(email, reset_url)

    return jsonify({"ok": True})


@app.route("/reset-password", methods=["GET", "POST"])
def reset_password():
    """
    GET  /reset-password?token=...  → show a reset form
    POST /reset-password            → JSON { "token": "...", "password": "..." }
    """
    if request.method == "GET":
        token = request.args.get("token", "")
        try:
            ts.loads(token, salt="password-reset", max_age=RESET_TOKEN_MAX_AGE)
        except (BadSignature, SignatureExpired):
            return render_template_string(_stub_page(
                "Link expired",
                "This password-reset link is invalid or has expired. "
                "Please request a new one from the sign-in page."
            ))
        return render_template_string(_reset_form_page(token))

    # POST — apply the new password
    data     = request.get_json(silent=True) or {}
    token    = data.get("token") or ""
    password = data.get("password") or ""

    if len(password) < 8:
        return jsonify({"ok": False, "error": "Password must be at least 8 characters."}), 400

    try:
        email = ts.loads(token, salt="password-reset", max_age=RESET_TOKEN_MAX_AGE)
    except SignatureExpired:
        return jsonify({"ok": False, "error": "Reset link has expired."}), 400
    except BadSignature:
        return jsonify({"ok": False, "error": "Invalid reset link."}), 400

    token_hash = hashlib.sha256(token.encode()).hexdigest()

    row = supabase.table("reset_tokens") \
        .select("*") \
        .eq("token_hash", token_hash) \
        .eq("used", False) \
        .execute()

    if not row.data:
        return jsonify({"ok": False, "error": "Reset link has already been used."}), 400

    user = get_user_by_email(email)
    if not user:
        return jsonify({"ok": False, "error": "Account not found."}), 400

    new_hash = bcrypt.generate_password_hash(password).decode("utf-8")

    supabase.table("users") \
        .update({"password_hash": new_hash}) \
        .eq("id", user["id"]) \
        .execute()

    supabase.table("reset_tokens") \
        .update({"used": True}) \
        .eq("token_hash", token_hash) \
        .execute()

    return jsonify({"ok": True})


@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("index"))


# ──────────────────────────────────────────────────────────────────────
# Routes — Google OAuth stub
# ──────────────────────────────────────────────────────────────────────

@app.route("/auth/google")
def auth_google():
    return render_template_string(_stub_page(
        "Google Sign-In",
        "Google OAuth is not yet configured. "
        "Please sign in with email and password."
    ))


# ──────────────────────────────────────────────────────────────────────
# Routes — protected
# ──────────────────────────────────────────────────────────────────────

@app.route("/dashboard")
@login_required
def dashboard():
    user = get_user_by_id(session["user_id"])
    if not user:
        session.clear()
        return redirect(url_for("index"))

    papers_res = supabase.table("papers") \
        .select("*") \
        .order("year", desc=True) \
        .limit(50) \
        .execute()

    papers = papers_res.data or []

    return render_template("dashboard.html", user=user, papers=papers)


# ──────────────────────────────────────────────────────────────────────
# Minimal HTML templates
# ──────────────────────────────────────────────────────────────────────

def _stub_page(title: str, body: str) -> str:
    return f"""<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>{title} — ThutoClub LS</title>
  <link href="https://fonts.googleapis.com/css2?family=Lora:wght@600&family=Source Sans+3:wght@400;500&display=swap" rel="stylesheet">
  <style>
    *, *::before, *::after {{ box-sizing: border-box; margin: 0; padding: 0; }}
    body {{ font-family: 'Source Sans 3', sans-serif; background: #FAFAF8; color: #1A1A18;
            display: flex; flex-direction: column; min-height: 100vh; }}
    nav {{ background: #fff; border-bottom: 1px solid #E8E6E0;
           display: flex; align-items: center; justify-content: space-between;
           padding: 0 2rem; height: 56px; }}
    .logo {{ font-family: 'Lora', serif; font-size: 18px; font-weight: 600;
             display: flex; align-items: center; gap: 8px;
             text-decoration: none; color: #1A1A18; }}
    .dot {{ width: 9px; height: 9px; border-radius: 50%; background: #0F6E56; }}
    main {{ flex: 1; display: flex; align-items: center; justify-content: center; padding: 3rem 1rem; }}
    .card {{ max-width: 500px; background: #fff; border: 1px solid #E8E6E0;
             border-radius: 10px; padding: 2rem; text-align: center; }}
    h1 {{ font-family: 'Lora', serif; font-size: 22px; margin-bottom: 0.75rem; }}
    p {{ color: #6B6A65; font-size: 15px; line-height: 1.6; }}
    a {{ color: #0F6E56; text-decoration: none; }}
    a:hover {{ text-decoration: underline; }}
    .back {{ display: inline-block; margin-top: 1.25rem; font-size: 14px; }}
  </style>
</head>
<body>
  <nav>
    <a class="logo" href="/"><span class="dot"></span>ThutoClub LS</a>
    <div style="display:flex;gap:12px">
      <a href="/subjects" style="font-size:14px;color:#6B6A65;text-decoration:none">Subjects</a>
      <a href="/years"    style="font-size:14px;color:#6B6A65;text-decoration:none">Years</a>
      <a href="/about"    style="font-size:14px;color:#6B6A65;text-decoration:none">About</a>
    </div>
  </nav>
  <main>
    <div class="card">
      <h1>{title}</h1>
      <p>{body}</p>
      <a class="back" href="/">← Back to sign in</a>
    </div>
  </main>
</body>
</html>"""


def _reset_form_page(token: str) -> str:
    return f"""<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Reset Password — ThutoClub LS</title>
  <link href="https://fonts.googleapis.com/css2?family=Lora:wght@600&family=Source+Sans+3:wght@400;500&display=swap" rel="stylesheet">
  <style>
    *, *::before, *::after {{ box-sizing: border-box; margin: 0; padding: 0; }}
    body {{ font-family: 'Source Sans 3', sans-serif; background: #FAFAF8; color: #1A1A18;
            display: flex; flex-direction: column; min-height: 100vh; }}
    nav {{ background: #fff; border-bottom: 1px solid #E8E6E0;
           display: flex; align-items: center; padding: 0 2rem; height: 56px; }}
    .logo {{ font-family: 'Lora', serif; font-size: 18px; font-weight: 600;
             display: flex; align-items: center; gap: 8px;
             text-decoration: none; color: #1A1A18; }}
    .dot {{ width: 9px; height: 9px; border-radius: 50%; background: #0F6E56; }}
    main {{ flex: 1; display: flex; align-items: center; justify-content: center; padding: 2rem 1rem; }}
    .card {{ width: 100%; max-width: 400px; background: #fff; border: 1px solid #E8E6E0;
             border-radius: 10px; overflow: hidden; }}
    .card-top {{ background: #E1F5EE; padding: 1.5rem 2rem; border-bottom: 1px solid #C2E8D8; }}
    .card-top h2 {{ font-family: 'Lora', serif; font-size: 20px; }}
    .card-top p  {{ font-size: 13px; color: #6B6A65; margin-top: 0.2rem; }}
    .card-body {{ padding: 1.5rem 2rem; }}
    label {{ display: block; font-size: 12px; font-weight: 500; color: #6B6A65;
             letter-spacing: 0.03em; text-transform: uppercase; margin-bottom: 5px; }}
    input {{ width: 100%; padding: 9px 12px; border: 1px solid #C8C5BC; border-radius: 6px;
             font-family: inherit; font-size: 14px; outline: none; background: #FAFAF8;
             transition: border-color .15s; }}
    input:focus {{ border-color: #0F6E56; box-shadow: 0 0 0 3px rgba(15,110,86,.08); }}
    .field {{ margin-bottom: 1rem; }}
    .err {{ font-size: 12px; color: #C0392B; margin-top: 4px; display: none; }}
    .banner {{ background: #FDF0EE; border: 1px solid #E8B4AF; border-radius: 6px;
               padding: 10px 12px; font-size: 13px; color: #C0392B;
               display: none; margin-bottom: 1rem; }}
    .success {{ background: #E1F5EE; border: 1px solid #C2E8D8; border-radius: 6px;
                padding: 10px 12px; font-size: 13px; color: #0a5a45;
                display: none; margin-bottom: 1rem; }}
    .btn {{ width: 100%; padding: 10px; background: #0F6E56; color: #fff;
            border: none; border-radius: 6px; font-family: inherit; font-size: 14px;
            font-weight: 500; cursor: pointer; transition: background .15s; }}
    .btn:hover {{ background: #0a5a45; }}
    .btn:disabled {{ opacity: .6; cursor: not-allowed; }}
  </style>
</head>
<body>
  <nav><a class="logo" href="/"><span class="dot"></span>ThutoClub LS</a></nav>
  <main>
    <div class="card">
      <div class="card-top">
        <h2>Choose a new password</h2>
        <p>Enter a password with at least 8 characters.</p>
      </div>
      <div class="card-body">
        <div class="banner" id="err-banner"></div>
        <div class="success" id="ok-banner">
          Password updated! <a href="/">Sign in →</a>
        </div>
        <div class="field">
          <label for="pw">New password</label>
          <input type="password" id="pw" placeholder="At least 8 characters">
          <div class="err" id="pw-err">Password must be at least 8 characters.</div>
        </div>
        <div class="field">
          <label for="pw2">Confirm password</label>
          <input type="password" id="pw2" placeholder="Repeat password">
          <div class="err" id="pw2-err">Passwords do not match.</div>
        </div>
        <button class="btn" id="submit-btn" onclick="doReset()">Set new password</button>
      </div>
    </div>
  </main>
  <script>
    async function doReset() {{
      const pw  = document.getElementById('pw').value;
      const pw2 = document.getElementById('pw2').value;
      document.querySelectorAll('.err').forEach(e => e.style.display = 'none');
      document.getElementById('err-banner').style.display = 'none';
      let ok = true;
      if (pw.length < 8)  {{ document.getElementById('pw-err').style.display  = 'block'; ok = false; }}
      if (pw !== pw2)      {{ document.getElementById('pw2-err').style.display = 'block'; ok = false; }}
      if (!ok) return;
      const btn = document.getElementById('submit-btn');
      btn.disabled = true;
      btn.textContent = 'Saving…';
      const res  = await fetch('/reset-password', {{
        method: 'POST',
        headers: {{ 'Content-Type': 'application/json' }},
        body: JSON.stringify({{ token: '{token}', password: pw }}),
      }});
      const data = await res.json();
      if (data.ok) {{
        document.getElementById('ok-banner').style.display = 'block';
        btn.style.display = 'none';
      }} else {{
        const banner = document.getElementById('err-banner');
        banner.textContent = data.error || 'Something went wrong.';
        banner.style.display = 'block';
        btn.disabled = false;
        btn.textContent = 'Set new password';
      }}
    }}
  </script>
</body>
</html>"""


# ──────────────────────────────────────────────────────────────────────
# Entry point
# ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
