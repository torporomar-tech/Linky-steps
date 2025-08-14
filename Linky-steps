# -*- coding: utf-8 -*-
"""
LinkySteps — Flask link shortener with multi-step interstitial pages (AdSense-ready placeholders).
- RTL Arabic UI, mobile-friendly.
- 3 stages: Home → Interstitial (20s) → Final Redirect.
- Multiple ad placeholders on each page. Replace placeholders AFTER AdSense approval.
- Compliant approach: no forcing/encouraging ad clicks; provide real content on interstitials.
"""

import os
import re
import sqlite3
import string
import random
from datetime import datetime
from urllib.parse import urlparse

from flask import Flask, g, redirect, render_template_string, request, url_for, abort, jsonify

APP_NAME = "LinkySteps"
DATABASE = os.path.join(os.path.dirname(__file__), "data.sqlite3")
CODE_LEN = 6

# Interstitial config
WAIT_SECONDS = 20  # per your request
INTERSTITIAL_STEPS = 1  # single interstitial page before final

app = Flask(__name__)
app.config["SECRET_KEY"] = os.getenv("SECRET_KEY", "dev-secret-change-me")

# ------------------- DB helpers -------------------

def get_db():
    if "db" not in g:
        g.db = sqlite3.connect(DATABASE)
        g.db.row_factory = sqlite3.Row
    return g.db

@app.teardown_appcontext
def close_db(_):
    db = g.pop("db", None)
    if db is not None:
        db.close()

def init_db():
    db = get_db()
    db.execute(
        """
        CREATE TABLE IF NOT EXISTS links (
            code TEXT PRIMARY KEY,
            target TEXT NOT NULL,
            created_at TEXT NOT NULL,
            title TEXT,
            clicks INTEGER NOT NULL DEFAULT 0
        );
        """
    )
    db.commit()

with app.app_context():
    init_db()

# ------------------- Utils -------------------

BASE62 = string.ascii_letters + string.digits

def is_valid_url(u: str) -> bool:
    try:
        p = urlparse(u)
        return p.scheme in {"http", "https"} and bool(p.netloc)
    except Exception:
        return False

def random_code(n=CODE_LEN) -> str:
    return "".join(random.choice(BASE62) for _ in range(n))

def unique_code():
    db = get_db()
    while True:
        c = random_code()
        row = db.execute("SELECT code FROM links WHERE code = ?", (c,)).fetchone()
        if row is None:
            return c

# ------------------- Templates -------------------

BASE_HTML = r"""
<!doctype html>
<html lang="ar" dir="rtl">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{ title or app_name }}</title>
  <meta name="description" content="موقع اختصار روابط بتصميم عصري وصفحات انتقال جاهزة للإعلانات.">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Cairo:wght@400;600;700&display=swap" rel="stylesheet">
  <style>
    :root { --bg:#0b1220; --card:#0f172a; --muted:#94a3b8; --fg:#e5e7eb; --accent:#22d3ee; --ring:#60a5fa; }
    * { box-sizing: border-box }
    body { margin:0; font-family:'Cairo',system-ui,Arial; background: radial-gradient(1200px 600px at 20% -10%, #12223d 0, transparent 60%), radial-gradient(1000px 400px at 80% -20%, #0c2a39 0, transparent 60%), linear-gradient(180deg,#0b1220,#0b1220 60%, #0f172a); color:var(--fg); }
    a { color:var(--accent); text-decoration:none }
    header { padding:24px 16px; border-bottom:1px solid rgba(255,255,255,.06) }
    .wrap { max-width: 920px; margin: 0 auto; padding: 16px }
    .brand { display:flex; align-items:center; gap:10px; font-weight:700 }
    .brand .logo { width:28px; height:28px }
    .grid { display:grid; gap:14px }
    .grid.two { grid-template-columns: 1fr 1fr }
    .card { background:rgba(15,23,42,.75); border:1px solid rgba(255,255,255,.06); border-radius:18px; padding:20px; box-shadow:0 10px 35px rgba(0,0,0,.35) }
    input, button, select { width:100%; padding:12px 14px; border-radius:12px; border:1px solid rgba(255,255,255,.1); background:#0a1020; color:var(--fg); outline:none }
    input:focus, select:focus { border-color: var(--ring); box-shadow:0 0 0 3px rgba(96,165,250,.15) }
    .btn { cursor:pointer; background:linear-gradient(135deg,#22d3ee,#60a5fa); font-weight:700 }
    .btn:disabled { opacity:.6; cursor:not-allowed }
    .muted { color:var(--muted) }
    .ctr { text-align:center }
    .space { height:14px }
    .ads { min-height:120px; border:1px dashed rgba(255,255,255,.18); border-radius:14px; display:flex; align-items:center; justify-content:center; font-size:14px; color:var(--muted); margin:10px 0 }
    footer { padding:24px 16px; color:var(--muted) }
    code { background: rgba(255,255,255,.06); padding:2px 6px; border-radius:6px }
  </style>
  <!-- Place once approved by AdSense -->
  <!-- ADSENSE_GLOBAL: <script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-XXXX" crossorigin="anonymous"></script> -->
</head>
<body>
  <header>
    <div class="wrap" style="display:flex; align-items:center; justify-content:space-between; gap:12px">
      <div class="brand">
        <svg class="logo" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path d="M7 7l10 10M7 17 17 7" stroke="#22d3ee" stroke-width="2" stroke-linecap="round"/>
        </svg>
        <span>{{ app_name }}</span>
        <span style="font-size:12px; background:rgba(255,255,255,.07); padding:4px 8px; border-radius:999px">اختصار روابط</span>
      </div>
      <nav class="muted"><a href="/">الرئيسية</a> · <a href="/privacy">الخصوصية</a></nav>
    </div>
  </header>
  <main class="wrap">
    {% block content %}{% endblock %}
  </main>
  <footer class="wrap">
    <div class="card">© {{ year }} {{ app_name }} — الرجاء عدم تحفيز المستخدمين على الضغط على الإعلانات.</div>
  </footer>
</body>
</html>
"""

INDEX_HTML = r"""
{% extends base %}
{% block content %}
  <div class="card">
    <h1 style="margin:0 0 8px">اختصر رابطك بأمان</h1>
    <p class="muted">ضع الرابط وسيتم إنشاء رابط مختصر يمر عبر صفحة انتظار ({{ wait_seconds }} ثانية) قبل الوصول للوجهة. في الصفحات أماكن لإعلانات AdSense (تضيفها بعد القبول).</p>
    <div class="space"></div>
    <!-- Ad placeholders (safe number: up to 2 on home) -->
    <div class="ads">مكان إعلان 1 — ضع وحدة AdSense بعد القبول
      <!-- ADSENSE_BLOCK: <ins class="adsbygoogle" style="display:block" data-ad-client="ca-pub-XXXX" data-ad-slot="YYYY" data-ad-format="auto" data-full-width-responsive="true"></ins><script>(adsbygoogle=window.adsbygoogle||[]).push({});</script> -->
    </div>
    <div class="space"></div>

    <form method="post" action="{{ url_for('create') }}" class="grid">
      <div>
        <label>الرابط الأصلي</label>
        <input type="url" name="target" placeholder="https://example.com/article" required>
      </div>
      <button class="btn">اختصر الرابط</button>
    </form>

    <div class="space"></div>
    <!-- Another optional ad -->
    <div class="ads">مكان إعلان 2 — (اختياري)</div>
  </div>

  <div class="space"></div>
  <div class="card">
    <h3>كيف يعمل؟</h3>
    <ol>
      <li>أدخل رابطك الأصلي.</li>
      <li>تحصل على رابط مختصر مثل <code>{{ request.host_url }}go/abc123</code>.</li>
      <li>يمر الزائر على صفحة انتظار بمحتوى موجز وإعلانات (بدون تحفيز النقر).</li>
      <li>ثم ينتقل تلقائيًا إلى الرابط الأصلي.</li>
    </ol>
  </div>
{% endblock %}
"""

INTERSTITIAL_HTML = r"""
{% extends base %}
{% block content %}
  <div class="card">
    <h1>الرجاء الانتظار {{ seconds }} ثانية</h1>
    <p class="muted">صفحة انتقال تحتوي على محتوى مختصر مفيد ومساحات إعلانية متوافقة مع السياسات.</p>

    <!-- Top ad -->
    <div class="ads">مساحة إعلان أعلى المحتوى</div>

    <div class="space"></div>
    <div class="card" style="background:#091227">
      <h3>معلومة سريعة</h3>
      <p class="muted">أضف هنا نصًا موجزًا يقدم فائدة للزائر (ملخص، نصيحة، روابط مفيدة). وجود محتوى فعلي مهم للامتثال لسياسات AdSense.</p>
    </div>

    <div class="space"></div>
    <!-- In-content ad(s) -->
    <div class="ads">مساحة إعلان داخلية 1</div>
    <div class="ads">مساحة إعلان داخلية 2 (اختياري)</div>

    <div class="space"></div>
    <div class="ctr">
      <button id="btnNext" class="btn" disabled>متابعة</button>
      <div class="muted" id="countdown"></div>
    </div>
  </div>

  <script>
    const seconds = {{ seconds|int }};
    const nextUrl = {{ next_url|tojson }};
    let s = seconds;
    const cd = document.getElementById('countdown');
    const btn = document.getElementById('btnNext');
    function tick(){
      s -= 1;
      if(s <= 0){ btn.disabled = false; cd.textContent = 'يمكنك المتابعة الآن'; return; }
      cd.textContent = 'متبقي ' + s + ' ثانية';
      setTimeout(tick, 1000);
    }
    setTimeout(tick, 1000);
    btn.addEventListener('click', ()=>{ window.location.href = nextUrl; });
  </script>
{% endblock %}
"""

FINAL_REDIRECT_HTML = r"""
{% extends base %}
{% block content %}
  <div class="card ctr">
    <h1>جاهز للتحويل</h1>
    <p class="muted">إذا لم يتم تحويلك تلقائيًا، اضغط الزر للانتقال للرابط الأصلي.</p>

    <div class="space"></div>
    <!-- Ad near action (keep single) -->
    <div class="ads">مساحة إعلان</div>

    <div class="space"></div>
    <a class="btn" href="{{ target }}">اذهب الآن</a>
  </div>
  <script>
    setTimeout(function(){ window.location.href = {{ target|tojson }} }, 1200);
  </script>
{% endblock %}
"""

PRIVACY_HTML = r"""
{% extends base %}
{% block content %}
  <div class="card">
    <h1>الخصوصية وملفات تعريف الارتباط</h1>
    <p>نستخدم ملفات تعريف الارتباط لأغراض تشغيل الموقع وقياس الزيارات فقط. قد تعرض صفحات الانتقال وحدات إعلانية من أطراف ثالثة (مثل Google AdSense) والتي قد تستخدم ملفات تعريف الارتباط وفق سياساتها.</p>
    <ul>
      <li>لا نخزن بيانات شخصية حساسة.</li>
      <li>تستطيع حذف الرابط المختصر عبر مراسلتنا بإثبات الملكية.</li>
    </ul>
  </div>
{% endblock %}
"""

# ------------------- Routes -------------------

@app.before_request
def _ensure_db():
    init_db()

@app.context_processor
def inject_globals():
    return {"app_name": APP_NAME, "year": datetime.utcnow().year, "base": BASE_HTML}

@app.get("/")
def index():
    return render_template_string(INDEX_HTML, wait_seconds=WAIT_SECONDS)

@app.post("/create")
def create():
    target = (request.form.get("target") or "").strip()
    if not is_valid_url(target):
        return render_template_string(INDEX_HTML, wait_seconds=WAIT_SECONDS), 400
    code = unique_code()
    db = get_db()
    db.execute(
        "INSERT OR REPLACE INTO links(code, target, created_at, title) VALUES(?,?,?,?)",
        (code, target, datetime.utcnow().isoformat(), None),
    )
    db.commit()
    short = url_for("go", code=code, _external=True)
    return render_template_string(
        r"""
        {% extends base %}
        {% block content %}
        <div class=card>
          <h2>تم إنشاء الرابط</h2>
          <p>انسخ الرابط المختصر وشاركه:</p>
          <input value="{{ short }}" readonly onclick="this.select()">
          <div class="space"></div>
          <a class="btn" href="/">إنشاء رابط آخر</a>
          <div class="space"></div>
          <p class="muted">يمر الزائر عبر صفحة انتظار قبل التحويل.</p>
        </div>
        {% endblock %}
        """,
        short=short,
    )

@app.get("/go/<code>")
def go(code):
    db = get_db()
    row = db.execute("SELECT * FROM links WHERE code=?", (code,)).fetchone()
    if not row:
        abort(404)
    return redirect(url_for("interstitial", code=code, step=1))

@app.get("/s/<code>/<int:step>")
def interstitial(code, step):
    db = get_db()
    row = db.execute("SELECT * FROM links WHERE code=?", (code,)).fetchone()
    if not row:
        abort(404)
    total = INTERSTITIAL_STEPS
    if step > total:
        return redirect(url_for("final", code=code))
    next_url = url_for("interstitial", code=code, step=step + 1) if step < total else url_for("final", code=code)
    return render_template_string(
        INTERSTITIAL_HTML,
        seconds=WAIT_SECONDS,
        next_url=next_url,
    )

@app.get("/final/<code>")
def final(code):
    db = get_db()
    row = db.execute("SELECT * FROM links WHERE code=?", (code,)).fetchone()
    if not row:
        abort(404)
    target = row["target"]
    db.execute("UPDATE links SET clicks = clicks + 1 WHERE code=?", (code,))
    db.commit()
    return render_template_string(FINAL_REDIRECT_HTML, target=target)

@app.get("/privacy")
def privacy():
    return render_template_string(PRIVACY_HTML)

@app.get("/api/info/<code>")
def api_info(code):
    db = get_db()
    row = db.execute("SELECT code, target, created_at, clicks FROM links WHERE code=?", (code,)).fetchone()
    if not row:
        return jsonify({"error": "not_found"}), 404
    return jsonify(dict(row))

@app.errorhandler(404)
def not_found(e):
    return render_template_string(
        r"""
        {% extends base %}
        {% block content %}
        <div class="card">
          <h2>الرابط غير موجود</h2>
          <p class="muted">تأكد من صحة الرابط المختصر.</p>
          <a class="btn" href="/">العودة للرئيسية</a>
        </div>
        {% endblock %}
        """
    ), 404

if __name__ == "__main__":
    port = int(os.getenv("PORT", 5000))
    app.run(host="0.0.0.0", port=port)
