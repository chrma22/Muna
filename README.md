import os, json, shutil, zipfile

# Base directory for the new gallery version
base = "/mnt/data/munitiondb_gallery"
os.makedirs(base, exist_ok=True)
os.makedirs(f"{base}/data", exist_ok=True)
os.makedirs(f"{base}/images", exist_ok=True)

# Create a sample image placeholder
# For now, we'll just create a dummy image file (since original image from earlier run is gone)
from PIL import Image, ImageDraw
img = Image.new("RGB", (400, 300), color=(80, 80, 80))
d = ImageDraw.Draw(img)
d.text((120,140), "AZ 23", fill=(255, 200, 0))
img_path = f"{base}/images/az23.jpg"
img.save(img_path)

# index.html
index_html = """<!doctype html>
<html lang="de">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Ammunitionsdatenbank</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div class="app">
    <aside class="sidebar">
      <div class="brand">
        <div class="brand-mark">💣</div>
        <div class="brand-text">
          <div class="brand-top">MuniDB</div>
          <div class="brand-bottom">Katalog</div>
        </div>
      </div>

      <div class="search">
        <input id="searchInput" type="search" placeholder="Suche..." autocomplete="off">
      </div>

      <nav class="nav">
        <div class="nav-section">Kategorien</div>
        <button class="nav-link active" data-filter="alle">Alle</button>
        <button class="nav-link" data-filter="Zünder">Zünder</button>
        <button class="nav-link" data-filter="Artillerie">Artillerie</button>
        <button class="nav-link" data-filter="Bomben">Bomben</button>
      </nav>

      <div class="footer-buttons">
        <button id="btnFeedback" class="btn ghost">Feedback</button>
      </div>
    </aside>

    <main class="content">
      <header class="content-header">
        <h1 id="pageTitle">Ammunitionsdatenbank</h1>
        <span id="countBadge" class="badge">0 Einträge</span>
      </header>

      <section class="list-and-detail">
        <div class="gallery" id="itemList"></div>
        <article class="detail" id="detailView">
          <p>Bitte einen Eintrag auswählen...</p>
        </article>
      </section>
    </main>
  </div>
  <script src="app.js"></script>
</body>
</html>
"""

# styles.css
styles_css = """:root{
  --bg:#0e0e0f;
  --panel:#1a1b1e;
  --panel-2:#202127;
  --text:#e9e9ed;
  --subtext:#b5b5be;
  --muted:#8c8c94;
  --accent:#ff9f1a;
  --border:#2a2b31;
}
*{box-sizing:border-box}
body,html{margin:0;padding:0;height:100%;font-family:sans-serif;background:var(--bg);color:var(--text)}
.app{display:grid;grid-template-columns:280px 1fr;height:100vh}
.sidebar{background:var(--panel);border-right:1px solid var(--border);padding:16px;display:flex;flex-direction:column;gap:16px}
.brand{display:flex;align-items:center;gap:10px}
.brand-mark{font-size:24px}
.brand-bottom{font-size:12px;color:var(--muted)}
.search input{width:100%;padding:8px;border-radius:6px;border:1px solid var(--border);background:var(--panel-2);color:var(--text)}
.nav{display:flex;flex-direction:column;gap:6px}
.nav-link{padding:8px;border:none;border-radius:6px;background:var(--panel-2);color:var(--text);cursor:pointer;text-align:left}
.nav-link.active,.nav-link:hover{background:var(--accent);color:#000}
.footer-buttons{margin-top:auto}
.btn{padding:8px;border-radius:6px;border:none;cursor:pointer}
.btn.ghost{background:transparent;color:var(--text);border:1px dashed var(--border)}
.content{display:flex;flex-direction:column;overflow:hidden}
.content-header{display:flex;justify-content:space-between;align-items:center;padding:10px 16px;background:var(--panel);border-bottom:1px solid var(--border)}
.badge{background:var(--panel-2);padding:4px 8px;border-radius:10px}
.list-and-detail{display:grid;grid-template-columns:1fr 400px;gap:12px;padding:12px;overflow:hidden;flex:1}
.gallery{display:grid;grid-template-columns:repeat(auto-fill,minmax(180px,1fr));gap:12px;overflow:auto;background:var(--panel);border:1px solid var(--border);border-radius:8px;padding:12px}
.card-item{background:var(--panel-2);border:1px solid var(--border);border-radius:8px;overflow:hidden;cursor:pointer;display:flex;flex-direction:column}
.card-thumb{aspect-ratio:4/3;display:flex;align-items:center;justify-content:center;background:#000}
.card-thumb img{width:100%;height:100%;object-fit:cover}
.card-body{padding:8px;display:flex;flex-direction:column;gap:4px}
.card-title{font-weight:bold;font-size:14px}
.card-sub{font-size:12px;color:var(--muted)}
.detail{background:var(--panel);border:1px solid var(--border);border-radius:8px;padding:12px;overflow:auto}
.detail h2{margin-top:0}
.detail figure{margin:0 0 12px 0}
.detail img{max-width:100%;border-radius:6px}
@media(max-width:1000px){
  .list-and-detail{grid-template-columns:1fr}
}
"""

# app.js
app_js = """(() => {
  const state = {data:[],filter:'alle',query:'',selectedId:null};
  const el = sel => document.querySelector(sel);
  const els = sel => Array.from(document.querySelectorAll(sel));
  function fmtCount(n){return `${n} Eintrag${n===1?'':'e'}`;}
  async function load(){
    const res = await fetch('data/items.json');
    state.data = await res.json();
    renderList();
  }
  function renderList(){
    const list = el('#itemList');
    list.innerHTML = '';
    const items = filtered();
    items.forEach(it => {
      const card = document.createElement('div');
      card.className='card-item';
      card.dataset.id = it.id;
      const thumb = document.createElement('div');
      thumb.className='card-thumb';
      if(it.image){
        const img = document.createElement('img');
        img.src = it.image; img.alt = it.title;
        thumb.appendChild(img);
      } else {
        thumb.textContent='Kein Bild';
        thumb.style.color='var(--muted)';
      }
      const body = document.createElement('div');
      body.className='card-body';
      const title = document.createElement('div');
      title.className='card-title';
      title.textContent = it.title;
      const sub = document.createElement('div');
      sub.className='card-sub';
      sub.textContent = `${it.category} • ${it.country||'Unbekannt'}`;
      body.appendChild(title); body.appendChild(sub);
      card.appendChild(thumb); card.appendChild(body);
      card.addEventListener('click', () => select(it.id));
      list.appendChild(card);
    });
    el('#countBadge').textContent = fmtCount(items.length);
  }
  function filtered(){
    const q = state.query.toLowerCase();
    return state.data.filter(it => {
      const matchCat = state.filter==='alle'||it.category===state.filter;
      if(!matchCat) return false;
      if(!q) return true;
      return [it.title,it.category,(it.tags||[]).join(' '),(it.details?.Material||'')]
        .join(' ').toLowerCase().includes(q);
    });
  }
  function select(id){
    state.selectedId=id;
    const it = state.data.find(x=>x.id===id);
    if(!it) return;
    const view = el('#detailView');
    view.innerHTML = `
      <h2>${it.title}</h2>
      ${it.image?`<figure><img src="${it.image}" alt="${it.title}"></figure>`:''}
      <p><strong>Kategorie:</strong> ${it.category}</p>
      <ul>${it.description.map(p=>`<li>${p}</li>`).join('')}</ul>
      <dl>${Object.entries(it.details||{}).map(([k,v])=>`<dt>${k}</dt><dd>${v}</dd>`).join('')}</dl>
    `;
  }
  el('#searchInput').addEventListener('input', e => {state.query=e.target.value;renderList();});
  els('.nav-link').forEach(btn=>{
    btn.addEventListener('click',()=>{
      els('.nav-link').forEach(b=>b.classList.remove('active'));
      btn.classList.add('active');
      state.filter=btn.dataset.filter;
      renderList();
    });
  });
  el('#btnFeedback').addEventListener('click',()=>alert('Feedback an: example@example.com'));
  load();
})();"""

# Sample items.json
items = [
  {
    "id":"az23",
    "title":"AZ 23",
    "category":"Zünder",
    "country":"Deutsch",
    "tags":["Artillerie","Zünder"],
    "image":"images/az23.jpg",
    "description":["Beispielbeschreibung Punkt 1","Punkt 2","Punkt 3"],
    "details":{"Material":"Messing","Verwendung":"Demo","Kaliber":"7,62 mm"}
  },
  {
    "id":"demo1",
    "title":"Demo Granate",
    "category":"Artillerie",
    "country":"Unbekannt",
    "tags":["Artillerie"],
    "image":"",
    "description":["Beschreibung folgt"],
    "details":{"Hinweis":"Nur zu Demozwecken"}
  }
]

# Write files
with open(f"{base}/index.html","w",encoding="utf-8") as f:f.write(index_html)
with open(f"{base}/styles.css","w",encoding="utf-8") as f:f.write(styles_css)
with open(f"{base}/app.js","w",encoding="utf-8") as f:f.write(app_js)
with open(f"{base}/data/items.json","w",encoding="utf-8") as f:json.dump(items,f,ensure_ascii=False,indent=2)

# Zip it
zip_path = "/mnt/data/munitiondb_gallery.zip"
with zipfile.ZipFile(zip_path,"w",zipfile.ZIP_DEFLATED) as z:
    for root,dirs,files in os.walk(base):
        for name in files:
            full = os.path.join(root,name)
            rel = os.path.relpath(full, base)
            z.write(full, arcname=f"munitiondb_gallery/{rel}")

zip_path
