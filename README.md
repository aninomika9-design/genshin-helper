# -*- coding: utf-8 -*-
import json
import re
import time
import difflib
import urllib.request
from pathlib import Path
from urllib.parse import urljoin

from bs4 import BeautifulSoup  # pip install beautifulsoup4
from flask import Flask, jsonify, render_template_string, send_from_directory

API = "https://gi.yatta.moe/api/v2"
AMBR_RU = f"{API}/ru/avatar"
ASSETS = "https://gi.yatta.moe/assets/UI"
CACHE_HOURS = 6

# ---------- WOTPACK (билды: оружие и артефакты) ----------
WP_HOST = "https://wotpack.ru"
WP_INDEX = f"{WP_HOST}/cat/drugie-2/genshin-impact/bildy-genshin-impact/"
WP_CACHE_HOURS = 72  # как часто перечитывать страницы билдов
BROWSER_UA = ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
              "(KHTML, like Gecko) Chrome/124.0 Safari/537.36")

app = Flask(__name__)
BASE_DIR = Path(__file__).resolve().parent
WP_CACHE_DIR = BASE_DIR / "wp_cache"
WP_CACHE_DIR.mkdir(exist_ok=True)

TRAVELER_PREFIXES = ("10000005", "10000007")
TRAVELER_ELEMENTS = [
    ("Fire", "Пиро"), ("Water", "Гидро"), ("Wind", "Анемо"), ("Electric", "Электро"),
    ("Grass", "Дендро"), ("Ice", "Крио"), ("Rock", "Гео"),
]


def get(url):
    req = urllib.request.Request(url, headers={"User-Agent": "GenshinHelper/0.6"})
    return json.load(urllib.request.urlopen(req, timeout=20))


def cached(name, url, force=False):
    """Скачивает JSON и кеширует в файл. Если сеть недоступна, берёт старый кеш."""
    p = BASE_DIR / name
    stale = force or not p.exists() or (time.time() - p.stat().st_mtime) / 3600 > CACHE_HOURS
    if stale:
        try:
            print("Обновляю", name)
            p.write_text(json.dumps(get(url), ensure_ascii=False), encoding="utf-8")
        except Exception:
            if not p.exists():
                raise
    return json.loads(p.read_text(encoding="utf-8"))


def is_traveler(cid):
    return str(cid).startswith(TRAVELER_PREFIXES)


def is_valid(cid, c):
    name = c.get("name") or ""
    if "(пробный)" in name or "(тест)" in name.lower():
        return False
    return True


def load_characters(force=False):
    raw = cached("chars_ambr.json", AMBR_RU, force)
    items = raw.get("data", {}).get("items", {})
    res = []
    trav_icons = {}
    for cid, c in items.items():
        if is_traveler(cid):
            icon = c.get("icon") or ""
            if icon:
                trav_icons.setdefault(c.get("element") or "None", icon)
                trav_icons.setdefault("any", icon)
            continue
        if not is_valid(cid, c):
            continue
        icon = c.get("icon") or ""
        res.append({
            "id": str(cid),
            "name": c.get("name") or str(cid),
            "el": c.get("element") or "None",
            "wp": c.get("weaponType") or "",
            "rare": int(c.get("rank") or 4),
            "icon": f"{ASSETS}/{icon}.png" if icon else "",
        })
    # Путешественник: по одному на каждую стихию
    for el, ru in TRAVELER_ELEMENTS:
        icon = trav_icons.get(el) or trav_icons.get("any") or ""
        res.append({
            "id": f"traveler-{el}",
            "name": f"Путешественник ({ru})",
            "el": el,
            "wp": "WEAPON_SWORD_ONE_HAND",
            "rare": 5,
            "icon": f"{ASSETS}/{icon}.png" if icon else "",
        })
    res.sort(key=lambda x: (-x["rare"], x["name"]))
    return res


# ---------- WOTPACK: парсер ----------
def wp_get_html(url):
    req = urllib.request.Request(url, headers={
        "User-Agent": BROWSER_UA,
        "Accept-Language": "ru-RU,ru;q=0.9,en;q=0.8",
        "Accept": "text/html,application/xhtml+xml",
    })
    return urllib.request.urlopen(req, timeout=25).read().decode("utf-8", "replace")


def wp_cached_html(name, url, hours):
    p = WP_CACHE_DIR / name
    stale = not p.exists() or (time.time() - p.stat().st_mtime) / 3600 > hours
    if stale:
        try:
            p.write_text(wp_get_html(url), encoding="utf-8")
        except Exception:
            if not p.exists():
                raise
    return p.read_text(encoding="utf-8")


def norm_ru(s):
    s = (s or "").lower().replace("ё", "е").replace("э", "е").replace("й", "и")
    return re.sub(r"[^a-zа-я0-9]", "", s)


def wp_index():
    """{'Ху Тао': 'https://wotpack.ru/...'} со страницы «Билды Genshin Impact»."""
    html = wp_cached_html("_index.html", WP_INDEX, 24)
    soup = BeautifulSoup(html, "html.parser")
    res = {}
    for a in soup.find_all("a", href=True):
        href = a["href"]
        if not href.startswith(WP_HOST) or "/cat/" in href or a.find("img") is None:
            continue
        name = a.get_text(" ", strip=True)
        if name and name not in res:
            res[name] = href
    return res


def wp_find_url(ch):
    idx = wp_index()
    if not idx:
        return None
    if ch["id"].startswith("traveler-"):
        el = norm_ru(ch["name"].split("(")[-1].strip(") "))
        for n, u in idx.items():
            nn = norm_ru(n)
            if "путешественник" in nn and el in nn:
                return u
        return None
    target = norm_ru(ch["name"])
    by_norm = {norm_ru(n): u for n, u in idx.items()}
    if target in by_norm:
        return by_norm[target]
    # совпадение по слову (например «Флинс» внутри длинного имени)
    toks = {t for t in re.split(r"\s+", ch["name"].lower().replace("ё", "е")) if len(t) >= 4}
    toks = {norm_ru(t) for t in toks if "путешественник" not in t}
    hits = []
    for n, u in idx.items():
        ntoks = {norm_ru(t) for t in re.split(r"\s+", n.lower()) if len(t) >= 4}
        if toks & ntoks:
            hits.append(u)
    if len(set(hits)) == 1:
        return hits[0]
    close = difflib.get_close_matches(target, list(by_norm), n=1, cutoff=0.8)
    return by_norm[close[0]] if close else None


STAR = "⭐"


def _is_stars(line):
    s = line.replace("\ufe0f", "").replace(" ", "")
    return bool(s) and set(s) <= {STAR}


def _text(cell):
    for br in cell.find_all("br"):
        br.replace_with("\n")
    t = cell.get_text("")
    t = re.sub(r"[ \t\u00a0]+", " ", t)
    t = re.sub(r"\s*\n\s*", "\n", t).strip()
    return t


def _img(tag):
    img = tag.find("img")
    if not img:
        return ""
    for k in ("data-lazy-src", "data-src", "src"):
        v = img.get(k) or ""
        if v and not v.startswith("data:"):
            return urljoin(WP_HOST, v)
    return ""


def _is_item_table(tb):
    first = tb.find("tr")
    if not first:
        return False
    return "название" in first.get_text(" ").lower() and len(first.find_all(["th", "td"])) >= 3


def _section_tables(soup, keywords, check):
    """Таблицы, лежащие под заголовком, где встречается одно из keywords."""
    for h in soup.find_all(["h2", "h3", "h4"]):
        t = h.get_text(" ", strip=True).lower()
        if not any(k in t for k in keywords):
            continue
        lvl = int(h.name[1])
        tables = []
        for el in h.find_all_next(["table", "h2", "h3", "h4"]):
            if el.name == "table":
                tables.append(el)
            elif int(el.name[1]) <= lvl:
                break
        good = [tb for tb in tables if check(tb)]
        if good:
            return good
    return []


def _parse_items(table):
    items = []
    for tr in table.find_all("tr"):
        tds = tr.find_all("td")
        if len(tds) < 3:
            continue
        c0, c1, c2 = tds[0], tds[1], tds[2]
        anchors = []
        for a in c0.find_all("a"):
            t = a.get_text(" ", strip=True)
            if t and t not in anchors:
                anchors.append(t)
        icon = _img(c0)
        url = ""
        a0 = c0.find("a", href=True)
        if a0 and a0["href"].startswith(WP_HOST) and "__trashed" not in a0["href"]:
            url = a0["href"]
        lines = [l.strip() for l in _text(c0).split("\n") if l.strip()]
        star_lines = [l for l in lines if _is_stars(l)]
        other = [l for l in lines if not _is_stars(l)]
        if not other or other[0].lower() == "название":
            continue
        name = " + ".join(anchors) if anchors else other[0]
        rest = other[1:] if len(anchors) <= 1 else []
        if rest and len(rest) % 2 == 0:
            stats = " · ".join(f"{k}: {v}" for k, v in zip(rest[::2], rest[1::2]))
        else:
            stats = " · ".join(rest)
        counts = sorted({l.replace("\ufe0f", "").count(STAR) for l in star_lines})
        items.append({
            "name": name,
            "icon": icon,
            "url": url,
            "rare": counts[0] if len(counts) == 1 else 0,
            "rarity": "–".join(f"{c}★" for c in counts) if len(counts) > 1 else "",
            "stats": stats,
            "effect": _text(c1),
            "why": _text(c2),
        })
    return items


def _parse_stats(soup):
    tables = _section_tables(
        soup, ["рекомендуемые значения"],
        lambda tb: "основная" in (tb.find("tr").get_text(" ").lower() if tb.find("tr") else ""))
    rows = []
    if tables:
        for tr in tables[0].find_all("tr"):
            tds = tr.find_all("td")
            if len(tds) < 3:
                continue
            rows.append({
                "slot": _text(tds[0]),
                "main": _text(tds[1]).replace("\n", " / "),
                "subs": _text(tds[2]).replace("\n", ", "),
            })
    return rows


def wp_build(ch):
    """Оружие и артефакты персонажа со страницы билда wotpack.ru (с кешем)."""
    url = wp_find_url(ch)
    if not url:
        return None
    safe = re.sub(r"[^a-zA-Z0-9_-]", "_", ch["id"])
    p = WP_CACHE_DIR / f"build_{safe}.json"
    if p.exists() and (time.time() - p.stat().st_mtime) / 3600 < WP_CACHE_HOURS:
        return json.loads(p.read_text(encoding="utf-8"))
    try:
        soup = BeautifulSoup(wp_get_html(url), "html.parser")
    except Exception:
        if p.exists():
            return json.loads(p.read_text(encoding="utf-8"))
        raise
    w_tables = _section_tables(soup, ["рейтинг оружия", "оружи"], _is_item_table)
    a_tables = _section_tables(soup, ["выбор артефактов", "артефакт"], _is_item_table)
    weapons = [it for tb in w_tables for it in _parse_items(tb)]
    artifacts = _parse_items(a_tables[0]) if a_tables else []
    other = [it for tb in a_tables[1:] for it in _parse_items(tb)]
    data = {"url": url, "weapons": weapons, "artifacts": artifacts,
            "other_artifacts": other, "stats": _parse_stats(soup)}
    if weapons or artifacts:  # пустой результат не кешируем
        p.write_text(json.dumps(data, ensure_ascii=False), encoding="utf-8")
    return data


@app.get("/api/build/<cid>")
def build_for(cid):
    ch = next((c for c in load_characters() if c["id"] == cid), None)
    if not ch:
        return jsonify(ok=False, error="Персонаж не найден"), 404
    try:
        data = wp_build(ch)
    except Exception as e:
        return jsonify(ok=False, error=f"Не удалось загрузить wotpack.ru: {e}"), 500
    if not data or not (data["weapons"] or data["artifacts"]):
        return jsonify(ok=False, error="На wotpack.ru нет билда для этого персонажа."), 404
    return jsonify(ok=True, **data)


@app.get("/api/all")
def all_chars():
    return jsonify(load_characters())


@app.get("/api/refresh")
def refresh():
    try:
        chars = load_characters(force=True)
        return jsonify(ok=True, count=len(chars))
    except Exception as e:
        return jsonify(ok=False, error=str(e)), 500


@app.get("/bg.jpg")
def bg():
    return send_from_directory(BASE_DIR, "bg.jpg")


@app.get("/")
def index():
    return render_template_string(HTML)


HTML = r"""<!doctype html><html lang="ru"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1"><title>Genshin Helper</title>
<style>
*{box-sizing:border-box}
body{margin:0;font-family:system-ui,sans-serif;background:#0a0a0c;color:#eee;display:flex;min-height:100vh}
nav{width:72px;background:#0d0d10ee;display:flex;flex-direction:column;align-items:center;padding:12px 0;gap:8px;flex-shrink:0;z-index:20;border-right:1px solid #222}
nav button{width:56px;height:56px;border:0;border-radius:12px;background:#1a1a1f;color:#aaa;cursor:pointer;font-size:11px;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:4px}
nav button.on{background:#c9a227;color:#111;font-weight:700}
nav button:hover{background:#2a2a33;color:#fff}
nav button span{font-size:18px}
.wrap{flex:1;display:flex;flex-direction:column;min-width:0;position:relative}
header{background:#0d0d10cc;padding:14px 24px;font-size:20px;font-weight:700;backdrop-filter:blur(8px);border-bottom:1px solid #222;z-index:10}
main{flex:1;overflow:auto;padding:16px;position:relative}
.panel{display:none;max-width:1200px;margin:0 auto;position:relative;z-index:1}
.panel.on{display:block}
.panel.bg-panel::before{
  content:"";position:fixed;inset:0 0 0 72px;z-index:0;pointer-events:none;
  background:url('/bg.jpg') center/cover no-repeat;
  filter:brightness(0.35) saturate(1.1);
}
.panel.bg-panel > *{position:relative;z-index:1}
h1{margin:0 0 16px;border-left:4px solid #c9a227;padding-left:14px;font-size:22px;text-shadow:0 2px 8px #000}
input,select,button.act{padding:9px 12px;border-radius:8px;border:0;font-size:15px;margin:3px;background:#1a1a1fee;color:#eee}
button.act{background:#6c5ce7;cursor:pointer}
button.big{font-size:17px;padding:12px 22px;background:#c9a227;color:#111;font-weight:700}
.box{background:#12141acc;border:1px solid #2c303a88;border-radius:12px;padding:14px;margin-bottom:14px;backdrop-filter:blur(10px)}
.row{display:flex;flex-wrap:wrap;align-items:center;gap:6px}
.chip{background:#1a1a1fee;border-radius:16px;padding:5px 13px;font-size:13px;cursor:pointer;user-select:none;border:1px solid #333}
.chip.on{background:#c9a227;color:#111;border-color:#c9a227;font-weight:600}
.lbl{color:#bbb;font-size:13px;margin:0 4px}
.grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(110px,1fr));gap:18px 8px;margin-top:12px}
.c{text-align:center;font-size:13px;cursor:pointer;position:relative}
.c img{width:84px;height:84px;border-radius:50%;border:3px solid #9b6fd6;background:#222;object-fit:cover;box-shadow:0 4px 12px #0008}
.c.r5 img{border-color:#f0a030}.c.dim{opacity:.35}
.c .ok{position:absolute;top:0;right:18px;background:#2ecc71;border-radius:50%;width:22px;height:22px;line-height:22px;font-size:14px;box-shadow:0 2px 6px #000}
.muted{color:#aaa}.team{background:#1d2027cc;border-radius:12px;padding:12px;margin-top:10px;backdrop-filter:blur(8px)}
#modal{display:none;position:fixed;inset:0;background:#000a;align-items:center;justify-content:center;z-index:50;padding:12px}
#modal .in{background:#1d2027;padding:22px;border-radius:14px;width:100%;max-width:680px;max-height:92vh;overflow:auto;border:1px solid #333}
#mapFrame{width:100%;height:calc(100vh - 80px);border:0;border-radius:12px;background:#111}
.calc-grid{display:grid;grid-template-columns:1fr 1fr;gap:16px}
@media(max-width:700px){.calc-grid{grid-template-columns:1fr}}
.calc-card{background:#12141acc;border-radius:12px;padding:18px;border:1px solid #2c303a88;backdrop-filter:blur(10px)}
.calc-card h3{margin:0 0 12px;font-size:16px}
.calc-card input{width:100%;margin:8px 0;padding:12px;font-size:18px}
.calc-res{margin-top:12px;font-size:18px;font-weight:600;color:#2ecc71}
.calc-res .rest{color:#aaa;font-size:14px;font-weight:400}
.party-card{background:#12141acc;border:1px solid #2a2e3888;border-radius:14px;padding:16px;margin-top:14px;backdrop-filter:blur(10px)}
.party-card h3{margin:0 0 12px;font-size:15px;color:#ddd}
.party-grid{display:grid;grid-template-columns:repeat(4,1fr);gap:10px}
@media(max-width:600px){.party-grid{grid-template-columns:repeat(2,1fr)}}
.party-slot{background:#0d0f14cc;border-radius:12px;padding:12px 8px;text-align:center;border:1px solid #2a2e38}
.party-slot img{width:72px;height:72px;border-radius:50%;border:3px solid #6c5ce7;background:#222;object-fit:cover}
.party-slot.r5 img{border-color:#f0a030}
.party-slot .nm{margin-top:8px;font-size:13px;font-weight:600}
.party-slot .rl{font-size:11px;color:#9aa;margin-top:2px}
.party-meta{margin-top:10px;font-size:13px;color:#9aa}
.party-score{color:#2ecc71;font-weight:700}
/* билд: оружие и артефакты */
.tabs{display:flex;gap:8px;margin:16px 0 10px;border-top:1px solid #333;padding-top:12px}
.tabs .chip{font-size:14px;padding:7px 16px}
.wtitle2{margin:14px 0 8px;font-size:14px;color:#c9a227}
.wnote{font-size:12px;color:#999;margin:0 0 8px}
.wp{display:flex;gap:12px;align-items:flex-start;background:#12141a;border:1px solid #2c303a;border-radius:10px;padding:10px;margin-bottom:8px}
.wp.best{border-color:#c9a227;background:#1c1a10}
.wp img{width:64px;height:64px;object-fit:contain;border-radius:8px;background:#222;flex-shrink:0}
.wp .body{min-width:0;flex:1}
.wp .wn{font-weight:700;font-size:14px}
.wp .wn a{color:#eee;text-decoration:none;border-bottom:1px dotted #777}
.wp .wn a:hover{color:#c9a227}
.wp .st{color:#f0a030;font-size:12px;letter-spacing:1px}
.wp .st.s4{color:#b48cf0}
.wp .wstat{font-size:12px;color:#9ab;margin-top:2px}
.wp .tag{display:inline-block;font-size:11px;padding:1px 8px;border-radius:10px;background:#333;color:#ccc;margin-left:6px;vertical-align:middle}
.wp.best .tag{background:#c9a227;color:#111;font-weight:700}
.wp details{margin-top:6px}
.wp summary{cursor:pointer;font-size:12px;color:#c9a227;user-select:none}
.wp .weff,.wp .why{font-size:13px;color:#bbb;margin-top:6px;white-space:pre-line;line-height:1.4}
.wp .weff{color:#d8d2b8}
.stt{width:100%;border-collapse:collapse;font-size:12px;margin-top:4px}
.stt td{border:1px solid #2c303a;padding:6px 8px;vertical-align:top}
.stt td:first-child{color:#c9a227;white-space:nowrap}
.src{font-size:11px;color:#777;margin-top:10px}
.src a{color:#c9a227}
</style></head><body>
<nav>
  <button class="on" onclick="show('chars')" id="btn-chars"><span>👤</span>Персы</button>
  <button onclick="show('party')" id="btn-party"><span>⚔️</span>Пати</button>
  <button onclick="show('map')" id="btn-map"><span>🗺</span>Карта</button>
  <button onclick="show('calc')" id="btn-calc"><span>🧮</span>Кальк</button>
</nav>
<div class="wrap">
<header>Genshin Helper</header>
<main>

<div class="panel on bg-panel" id="panel-chars">
<h1>ПЕРСОНАЖИ</h1>
<div class="box">
 <div class="row"><input id="q" placeholder="Поиск по имени..." oninput="render()" style="flex:1;min-width:180px">
 <span class="lbl">Стихия:</span><span id="chEl" class="row"></span></div>
 <div class="row"><span class="lbl">Оружие:</span><span id="chWp" class="row"></span>
 <span class="lbl">Редкость:</span><span id="chR" class="row"></span>
 <span class="chip" id="chOwn" onclick="F.own=!F.own;render()">Только мои</span>
 <span class="chip" onclick="reset()">Сбросить</span>
 <button class="act" onclick="refreshChars()" style="background:#2ecc71;color:#111">Обновить</button>
 </div>
</div>
<p class="muted">Нажми на персонажа → «Есть у меня», роль, лучшее оружие и артефакты. Отмеченные персонажи попадут в подбор пати.</p>
<div id="cnt" class="muted"></div><div id="grid" class="grid"></div>
</div>

<div class="panel bg-panel" id="panel-party">
<h1>ПОДБОР ПАТИ</h1>
<div class="box">
  <div class="row">
    <span class="lbl">Главный персонаж:</span>
    <select id="mainChar" style="min-width:200px"></select>
    <button class="act big" onclick="buildParty()">Подобрать пати</button>
  </div>
  <p class="muted" style="margin:10px 0 0">Отметь своих персонажей во вкладке «Персы». Подбор только из них.</p>
</div>
<div id="partyOut"></div>
</div>

<div class="panel" id="panel-map">
<h1>КАРТА ТЕЙВАТА</h1>
<p class="muted" style="margin-bottom:10px">Официальная карта Hoyolab. Если не грузится —
<a href="https://act.hoyolab.com/ys/app/interactive-map/index.html?lang=ru-ru&bbs_theme=dark" target="_blank" style="color:#c9a227">открой в новой вкладке</a>.</p>
<iframe id="mapFrame" src="https://act.hoyolab.com/ys/app/interactive-map/index.html?lang=ru-ru&bbs_theme=dark&bbs_presentation_style=no_header" allowfullscreen></iframe>
</div>

<div class="panel bg-panel" id="panel-calc">
<h1>КАЛЬКУЛЯТОР КРУТОК</h1>
<p class="muted">1 крутка = 160 примогемов / кристаллов</p>
<div class="calc-grid">
  <div class="calc-card">
    <h3>Кристаллы → крутки</h3>
    <input type="number" id="crystals" placeholder="Например 1600" min="0" oninput="calcFromCrystals()">
    <div class="calc-res" id="resCrystals">—</div>
  </div>
  <div class="calc-card">
    <h3>Крутки → кристаллы</h3>
    <input type="number" id="pulls" placeholder="Например 10" min="0" oninput="calcFromPulls()">
    <div class="calc-res" id="resPulls">—</div>
  </div>
</div>
<div class="box" style="margin-top:16px">
  <b>Подсказки</b>
  <ul class="muted">
    <li>160 примогемов = 1 крутка</li>
    <li>90 круток ≈ гарант 5★ (~14 400)</li>
    <li>180 круток ≈ гарант нужного 5★ при 50/50</li>
  </ul>
</div>
</div>

</main>
</div>

<div id="modal" onclick="if(event.target==this)this.style.display='none'"><div class="in" id="mbody"></div></div>

<script>
const ELS=[["Fire","Пиро"],["Water","Гидро"],["Wind","Анемо"],["Electric","Электро"],["Grass","Дендро"],["Ice","Крио"],["Rock","Гео"]];
const WPS=[["WEAPON_SWORD_ONE_HAND","Одноручный"],["WEAPON_CLAYMORE","Двуручный"],["WEAPON_POLE","Древковое"],["WEAPON_CATALYST","Катализатор"],["WEAPON_BOW","Лук"]];
const RS=[["5","5★"],["4","4★"]];
const ROLES={dps:"DPS",sub:"Саб-DPS",sup:"Поддержка",heal:"Лечение"};
let all=[], mine=JSON.parse(localStorage.getItem("mine2")||"{}");
const F={el:new Set(),wp:new Set(),r:new Set(),own:false};
const save=()=>localStorage.setItem("mine2",JSON.stringify(mine));
const esc=s=>String(s).replace(/[&<>"]/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;"}[c]));
const owned=c=>mine[c.id]&&mine[c.id].owned;
const role=c=>(mine[c.id]&&mine[c.id].role)||(c.rare==5?"dps":"sub");
let openedId=null;

function show(name){
  document.querySelectorAll(".panel").forEach(p=>p.classList.remove("on"));
  document.querySelectorAll("nav button").forEach(b=>b.classList.remove("on"));
  document.getElementById("panel-"+name).classList.add("on");
  document.getElementById("btn-"+name).classList.add("on");
  if(name==="party") fillMainSelect();
}

function calcFromCrystals(){
  const n=Math.max(0,parseInt(document.getElementById("crystals").value)||0);
  const pulls=Math.floor(n/160), rest=n%160;
  document.getElementById("resCrystals").innerHTML=
    pulls+" круток"+(rest?`<div class="rest">остаток ${rest} (не хватает ${160-rest} до ещё одной)</div>`:"");
}
function calcFromPulls(){
  const n=Math.max(0,parseInt(document.getElementById("pulls").value)||0);
  document.getElementById("resPulls").innerHTML=(n*160).toLocaleString("ru-RU")+" кристаллов / примогемов";
}

function tog(t,k){const s=F[t];s.has(k)?s.delete(k):s.add(k);render()}
function reset(){F.el.clear();F.wp.clear();F.r.clear();F.own=false;document.getElementById("q").value="";render()}
function chips(id,arr,t){
  document.getElementById(id).innerHTML=arr.map(([k,l])=>
    `<span class="chip ${F[t].has(k)?"on":""}" onclick="tog('${t}','${k}')">${l}</span>`
  ).join("");
}

function render(){
  chips("chEl",ELS,"el");chips("chWp",WPS,"wp");chips("chR",RS,"r");
  document.getElementById("chOwn").className="chip"+(F.own?" on":"");
  const q=document.getElementById("q").value.toLowerCase(), any=all.some(owned);
  const list=all.filter(c=>c.name.toLowerCase().includes(q)&&(!F.el.size||F.el.has(c.el))&&(!F.wp.size||F.wp.has(c.wp))&&(!F.r.size||F.r.has(String(c.rare)))&&(!F.own||owned(c)));
  document.getElementById("cnt").textContent=list.length+" из "+all.length+" · отмечено своих: "+all.filter(owned).length;
  document.getElementById("grid").innerHTML=list.map(c=>
    `<div class="c r${c.rare} ${any&&!owned(c)?"dim":""}" onclick="openC('${c.id}')">
      <img src="${esc(c.icon)}" onerror="this.style.visibility='hidden'">
      ${owned(c)?'<div class="ok">✓</div>':""}
      <div>${esc(c.name)}</div>
    </div>`
  ).join("");
}

function fillMainSelect(){
  const sel=document.getElementById("mainChar");
  const mineList=all.filter(owned);
  const cur=sel.value;
  sel.innerHTML=mineList.length
    ? mineList.map(c=>`<option value="${c.id}">${esc(c.name)} (${ROLES[role(c)]})</option>`).join("")
    : '<option value="">Сначала отметь персонажей</option>';
  if(cur) sel.value=cur;
}

/* ---------- оружие и артефакты ---------- */
function itemHtml(w,tagFirst,tagRest,isFirst){
  const stars = w.rare ? `<div class="st s${w.rare}">${"★".repeat(w.rare)}</div>`
              : (w.rarity ? `<div class="st s4">${esc(w.rarity)}</div>` : "");
  const tag = isFirst ? (tagFirst?`<span class="tag">${tagFirst}</span>`:"") : (tagRest?`<span class="tag">${tagRest}</span>`:"");
  const nm = w.url ? `<a href="${esc(w.url)}" target="_blank" rel="noopener">${esc(w.name)}</a>` : esc(w.name);
  const more = (w.effect||w.why) ? `<details><summary>Подробнее</summary>
      ${w.effect?`<div class="weff">${esc(w.effect)}</div>`:""}
      ${w.why?`<div class="why">${esc(w.why)}</div>`:""}
    </details>` : "";
  return `<div class="wp ${isFirst&&tagFirst?"best":""}">
    <img src="${esc(w.icon||"")}" onerror="this.style.visibility='hidden'">
    <div class="body">
      <div class="wn">${nm}${tag}</div>
      ${stars}
      ${w.stats?`<div class="wstat">${esc(w.stats)}</div>`:""}
      ${more}
    </div>
  </div>`;
}

function weaponsHtml(j){
  if(!j.ok) return `<p class="wnote">${esc(j.error||"Ошибка загрузки")}</p>`;
  if(!j.weapons.length) return '<p class="wnote">Рекомендаций по оружию не нашёл.</p>';
  return j.weapons.map((w,i)=>itemHtml(w,"Лучший выбор","Альтернатива",i===0)).join("")+srcHtml(j);
}

function artifactsHtml(j){
  if(!j.ok) return `<p class="wnote">${esc(j.error||"Ошибка загрузки")}</p>`;
  if(!j.artifacts.length&&!j.other_artifacts.length)
    return '<p class="wnote">Рекомендаций по артефактам не нашёл.</p>';
  let h = j.artifacts.map((a,i)=>itemHtml(a,"Лучший выбор","Альтернатива",i===0)).join("");
  if(j.other_artifacts.length)
    h += `<div class="wtitle2">Другие сеты (с оговорками)</div>`+j.other_artifacts.map(a=>itemHtml(a,"","",false)).join("");
  if(j.stats&&j.stats.length)
    h += `<div class="wtitle2">Главные статы и подстаты</div><table class="stt">`+
      j.stats.map(r=>`<tr><td>${esc(r.slot)}</td><td>${esc(r.main)}</td><td>${esc(r.subs)}</td></tr>`).join("")+`</table>`;
  return h+srcHtml(j);
}

function srcHtml(j){
  return j.url
    ? `<div class="src">Источник: <a href="${esc(j.url)}" target="_blank" rel="noopener">wotpack.ru</a> — полный гайд по персонажу</div>` : "";
}

function tab(t){
  document.getElementById("wbox").style.display=t==="w"?"block":"none";
  document.getElementById("abox").style.display=t==="a"?"block":"none";
  document.getElementById("tw").classList.toggle("on",t==="w");
  document.getElementById("ta").classList.toggle("on",t==="a");
}

async function openC(id){
  const c=all.find(x=>x.id==id), m=mine[id]||{};
  openedId=id;
  document.getElementById("mbody").innerHTML=
    `<h3 style="margin-top:0">${esc(c.name)} ${c.rare}★</h3>
     <label><input type="checkbox" ${m.owned?"checked":""} onchange="setOwn('${id}',this.checked)"> Есть у меня</label><br>
     Роль: <select onchange="setRole('${id}',this.value)">
       ${Object.entries(ROLES).map(([k,v])=>`<option value="${k}" ${role(c)==k?"selected":""}>${v}</option>`).join("")}
     </select>
     <div class="tabs">
       <span class="chip on" id="tw" onclick="tab('w')">⚔ Оружие</span>
       <span class="chip" id="ta" onclick="tab('a')">🛡 Артефакты</span>
     </div>
     <div id="wbox" class="muted">Загрузка…</div>
     <div id="abox" class="muted" style="display:none">Загрузка…</div>
     <br><button class="act" onclick="document.getElementById('modal').style.display='none'">Закрыть</button>`;
  document.getElementById("modal").style.display="flex";
  let j;
  try{
    j=await (await fetch("/api/build/"+encodeURIComponent(id))).json();
  }catch(e){ j={ok:false,error:"Не удалось загрузить: "+e}; }
  if(openedId!==id) return;
  const wb=document.getElementById("wbox"), ab=document.getElementById("abox");
  if(wb) wb.innerHTML=weaponsHtml(j);
  if(ab) ab.innerHTML=artifactsHtml(j);
}
function setOwn(id,v){mine[id]=Object.assign(mine[id]||{},{owned:v});save();render();fillMainSelect()}
function setRole(id,v){mine[id]=Object.assign(mine[id]||{},{role:v});save();render();fillMainSelect()}

async function refreshChars(){
  const btn=event.target; btn.disabled=true; btn.textContent="Обновляю...";
  try{
    const j=await (await fetch("/api/refresh")).json();
    if(!j.ok){alert(j.error||"Ошибка");return}
    all=await (await fetch("/api/all")).json();
    render(); fillMainSelect();
    alert("Готово! Персонажей: "+all.length);
  }catch(e){alert("Ошибка: "+e)}
  finally{btn.disabled=false;btn.textContent="Обновить"}
}

const REACT={
  "Fire|Water":[3,"Испарение"],"Electric|Fire":[2,"Перегрузка"],"Ice|Water":[3,"Заморозка"],
  "Electric|Water":[2,"Электро-заряд"],"Electric|Grass":[3,"Ускорение"],"Grass|Water":[2,"Цветение"],
  "Fire|Grass":[2,"Горение"],"Fire|Ice":[2,"Таяние"],"Electric|Ice":[2,"Сверхпроводник"]
};
function pair(a,b){
  const k=[a.el,b.el].sort().join("|");
  if(REACT[k]) return REACT[k];
  const w=a.el==="Wind"?b:b.el==="Wind"?a:null;
  if(w && !["Rock","Grass","Wind"].includes(w.el)) return [2,"Рассеяние"];
  if(a.el==="Rock"||b.el==="Rock") return [1,"Кристаллизация"];
  return [0,""];
}

function scoreTeam(t){
  const roles=t.map(role);
  const dps=roles.filter(r=>r==="dps").length;
  if(dps<1||dps>2) return null;
  let s=0; const rs=new Set();
  for(let i=0;i<4;i++) for(let j=i+1;j<4;j++){
    const p=pair(t[i],t[j]); s+=p[0]; if(p[1]) rs.add(p[1]);
  }
  s += roles.includes("heal") ? 3 : -2;
  if(roles.includes("sup")) s += 2;
  if(roles.includes("sub")) s += 1;
  if(new Set(t.map(x=>x.el)).size===1) s -= 3;
  if(rs.size===0) s -= 2;
  if(t.some(x=>x.el==="Rock") || roles.includes("sup")) s+=2; else s-=1;
  return {s, rs:[...rs], roles};
}

function buildParty(){
  const out=document.getElementById("partyOut");
  const mainId=document.getElementById("mainChar").value;
  const pool=all.filter(owned);
  if(pool.length<4){
    out.innerHTML='<div class="team">Отметь минимум 4 персонажей во вкладке «Персы».</div>';
    return;
  }
  if(!mainId){
    out.innerHTML='<div class="team">Выбери главного персонажа.</div>';
    return;
  }
  const main=pool.find(c=>c.id===mainId);
  if(!main){
    out.innerHTML='<div class="team">Главный персонаж не отмечен как «есть».</div>';
    return;
  }
  const others=pool.filter(c=>c.id!==mainId);
  const res=[];
  const n=others.length;
  for(let a=0;a<n;a++) for(let b=a+1;b<n;b++) for(let c=b+1;c<n;c++){
    const t=[main, others[a], others[b], others[c]];
    const sc=scoreTeam(t);
    if(!sc) continue;
    res.push({t, ...sc});
  }
  res.sort((x,y)=>y.s-x.s);
  const top=res.slice(0,5);
  if(!top.length){
    out.innerHTML='<div class="team">Подходящих пати не найдено. Поменяй роли у персонажей.</div>';
    return;
  }
  const maxS=Math.max(...top.map(r=>r.s),1);
  out.innerHTML=`<p class="muted">Главный: <b>${esc(main.name)}</b> · из ${pool.length} твоих</p>`+
    top.map((r,i)=>{
      const pct=Math.min(100,Math.round(r.s/Math.max(maxS,8)*100));
      const slots=r.t.map(c=>`
        <div class="party-slot ${c.rare==5?"r5":""}">
          <img src="${esc(c.icon)}" onerror="this.style.visibility='hidden'">
          <div class="nm">${esc(c.name)}</div>
          <div class="rl">${ROLES[role(c)]}</div>
        </div>`).join("");
      return `<div class="party-card">
        <h3>Вариант ${i+1} · <span class="party-score">${pct}%</span></h3>
        <div class="party-grid">${slots}</div>
        <div class="party-meta">Реакции: ${r.rs.join(", ")||"нет"}</div>
      </div>`;
    }).join("");
}

fetch("/api/all").then(r=>r.json()).then(j=>{all=j;render();fillMainSelect()});
</script></body></html>"""

if __name__ == "__main__":
    print("Открой http://localhost:5000")
    print("Положи картинку как bg.jpg рядом с этим файлом")
    app.run(host="0.0.0.0", port=5000)
