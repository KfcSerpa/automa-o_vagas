# Job Finder Automation — Python + APIs (Remotive, RemoteOK, Adzuna, Jooble)

Automação para buscar, filtrar, priorizar e organizar vagas de **Dados / BI / DS Jr**.  
Consulta **APIs oficiais**, aplica filtros (Brasil / RMSP / remoto), remove duplicidades, calcula **score de relevância** e exporta um **Excel formatado** com links diretos para candidatura.

---

## Stack
- Python 3.10+
- Requests, Pandas
- XlsxWriter (export Excel)
- Dotenv (variáveis de ambiente)
- **APIs**: Remotive, RemoteOK, Adzuna, Jooble

---

## Arquitetura (resumo do pipeline)
APIs → Python (Requests + Pandas) → Filtros (área, local, remoto) →  
De-duplicação + Score → Excel com links prontos

---

## Score de relevância (como prioriza)
- **Cargo** citado (Analista de Dados, BI, Data Analyst, DS Jr)
- **Tecnologias** (SQL, Power BI, Python, etc.)
- **Local** (RMSP) ou **Remoto**
- **Recência** (últimos 7 dias com pesos maiores para vagas mais novas)
- **Fonte** (bônus leve para APIs mais confiáveis)

Resultado: as vagas mais aderentes sobem automaticamente no ranking.

---

## Pré-requisitos
- Python 3.10+ instalado
- Chaves das APIs (opcional, mas recomendado):
  - Adzuna: `ADZUNA_APP_ID`, `ADZUNA_APP_KEY`
  - Jooble: `JOOBLE_KEY`

> Remotive e RemoteOK funcionam sem chave. Habilitar Adzuna/Jooble aumenta bastante o volume no Brasil.

---

## Instalação

1. Clone este repositório e entre na pasta:
   ```bash
   git clone https://github.com/<seu-user>/<seu-repo>.git
   cd <seu-repo>

## Código completo do projeto

📌 Cell 1 — Dependências
%pip install -q --upgrade pip
%pip install -q python-dotenv pandas requests XlsxWriter

📌 Cell 2 — Pipeline completo:

# jobs_scraper (Jupyter) — v3.4 BR-first + RMSP + Adzuna/Jooble/Remotive/RemoteOK

import os, re, time, unicodedata, urllib.parse, requests, pandas as pd, warnings, http.client, json
from datetime import datetime, timedelta, timezone
from pathlib import Path
from pandas.api.types import is_datetime64_any_dtype
from pandas.core.dtypes.dtypes import DatetimeTZDtype

warnings.filterwarnings("ignore", category=UserWarning)
warnings.filterwarnings("ignore", category=DeprecationWarning)

# Load .env (current folder or user home)
loaded_paths = []
try:
    from dotenv import load_dotenv
    for p in [Path.cwd() / ".env", Path.home() / ".env"]:
        if p.exists():
            load_dotenv(dotenv_path=p, override=True)
            loaded_paths.append(str(p))
except Exception:
    pass

ADZUNA_APP_ID = os.getenv("ADZUNA_APP_ID", "").strip()
ADZUNA_APP_KEY = os.getenv("ADZUNA_APP_KEY", "").strip()
JOOBLE_KEY    = os.getenv("JOOBLE_KEY", "").strip()

print(f".env carregado de: {loaded_paths}" if loaded_paths else "Nenhum .env encontrado.")
print("Adzuna key:", "OK" if ADZUNA_APP_KEY else "faltando")
print("Jooble key:", "OK" if JOOBLE_KEY else "faltando")

ADZUNA_COUNTRY = "br"
DAYS = 7
MAX_PAGES_ADZUNA = 5
MAX_ROWS = 1000

SEARCH_TERMS = [
    "Analista de Dados", "Analista de BI",
    "Business Intelligence", "Data Analyst",
    "Cientista de Dados Jr", "Data Scientist Jr"
]

TITLE_REGEX = re.compile(
    r"(?:analista\s*de\s*dados|analista\s*de\s*bi|business\s*intelligence|"
    r"data\s*analyst|cientista\s*de\s*dados\s*jr|data\s*scientist\s*jr)",
    re.I
)

TARGET_CITIES = {"sao paulo", "sp", "barueri", "osasco", "carapicuiba", "jandira"}
BR_TOKENS = {"brasil", "brazil", "br"}

REMOTE_TOKENS = {"remote", "remoto", "anywhere", "global", "worldwide", "work from home", "home office"}
HYBRID_TOKENS = {"hybrid", "híbrido", "hibrido"}

# Helpers
def now_utc(): return datetime.now(timezone.utc)
def normalize_url(u): return urllib.parse.urlunsplit((*(urllib.parse.urlsplit(u)[:3]), "", ""))

def safe_to_dt(x):
    try: return pd.to_datetime(x, utc=True, errors="coerce")
    except: return pd.NaT

def filter_days(df, col="posted_at", days=DAYS):
    cut = now_utc() - timedelta(days=days)
    df[col] = df[col].map(safe_to_dt)
    return df[df[col] >= cut]

def strip_tz(df):
    for c in df.columns:
        if is_datetime64_any_dtype(df[c]) and isinstance(df[c].dtype, DatetimeTZDtype):
            df[c] = df[c].dt.tz_localize(None)
    return df

def text(s): return unicodedata.normalize("NFKD", str(s).lower()).encode("ascii","ignore").decode()

def is_remote_blob(s): return any(t in text(s) for t in REMOTE_TOKENS)
def is_hybrid_blob(s): return any(t in text(s) for t in HYBRID_TOKENS)
def is_br_blob(s): return any(t in text(s).split() for t in BR_TOKENS) or "sao paulo" in text(s)

def city_match(s): 
    t=text(s)
    return any(c in t for c in TARGET_CITIES) or "sao paulo, sp" in t

def mode_class(row):
    blob=f"{row.get('title','')} {row.get('description','')} {row.get('location','')}"
    if is_remote_blob(blob): return "remote"
    if is_hybrid_blob(blob): return "hybrid"
    return "onsite"

def loc_ok(mode, loc, company, title, desc):
    if mode=="remote": return True
    ctx=f"{loc} {company} {title} {desc}"
    return is_br_blob(ctx) and city_match(loc or ctx)

def days_ago(ts):
    if pd.isna(ts): return 999
    return (now_utc() - ts).days

# Score rules
ROLE={"analista de dados":5,"data analyst":5,"analista de bi":5,"business intelligence":4,"cientista de dados jr":4,"data scientist jr":4,"cientista de dados":3,"data scientist":3}
STACK={"sql":3,"power bi":3,"powerbi":3,"python":2,"pandas":1,"spark":1,"aws":1,"etl":1,"dashboards":1,"excel":1}
CITY={c:3 for c in TARGET_CITIES}

def score(row):
    blob=f"{row.title} {row.description} {row.location}".lower()
    role=sum(w for k,w in ROLE.items() if k in blob)
    tech=sum(w for k,w in STACK.items() if k in blob)
    city=sum(w for k,w in CITY.items() if k in blob)
    mode=2 if row.mode=="remote" else 1 if row.mode=="hybrid" else 0
    rec=3 if days_ago(row.posted_at)<=1 else 2 if days_ago(row.posted_at)<=3 else 1 if days_ago(row.posted_at)<=7 else 0
    src=1 if row.source.lower() in {"adzuna","jooble"} else 0
    return role*2 + tech + city + mode + rec + src

def terms_hit(text):
    return [t for t in SEARCH_TERMS if t.lower() in str(text).lower()]

# Fetchers
def fetch_remotive():
    try:
        r=requests.get("https://remotive.com/api/remote-jobs",timeout=30)
        jobs=r.json().get("jobs",[])
        rows=[{
            "title":j["title"],"company":j["company_name"],"location":j.get("candidate_required_location","Remote"),
            "url":j["url"],"posted_at":j["publication_date"],"salary":j.get("salary"),"description":j.get("description","")
        } for j in jobs if TITLE_REGEX.search(j["title"]) or TITLE_REGEX.search(j.get("description",""))]
    except: return pd.DataFrame()
    df=pd.DataFrame(rows)
    return filter_days(df,"posted_at")

def fetch_remoteok():
    try:
        r=requests.get("https://remoteok.com/api",headers={"User-Agent":"Mozilla"},timeout=30)
        jobs=r.json()[1:]
        rows=[]
        for j in jobs:
            t=j.get("position") or j.get("title") or ""
            d=j.get("description","")
            if TITLE_REGEX.search(t) or TITLE_REGEX.search(d):
                p=j.get("date") or (datetime.fromtimestamp(j["epoch"],tz=timezone.utc).isoformat() if j.get("epoch") else None)
                loc=j.get("location") or j.get("location_raw") or "Remote"
                if isinstance(loc,list):loc=", ".join(loc)
                rows.append({
                    "title":t,"company":j.get("company"),
                    "location":loc,"url":j.get("url") or j.get("apply_url"), 
                    "posted_at":p,"salary":None,"description":d
                })
    except: return pd.DataFrame()
    df=pd.DataFrame(rows)
    return filter_days(df,"posted_at")

def fetch_adzuna():
    if not (ADZUNA_APP_ID and ADZUNA_APP_KEY): return pd.DataFrame()
    res=[]
    q=urllib.parse.quote(" OR ".join(SEARCH_TERMS))
    for p in range(1,MAX_PAGES_ADZUNA+1):
        url=f"https://api.adzuna.com/v1/api/jobs/br/search/{p}?app_id={ADZUNA_APP_ID}&app_key={ADZUNA_APP_KEY}&what={q}&results_per_page=50&max_days_old={DAYS}"
        try:
            r=requests.get(url,timeout=30)
            for j in r.json().get("results",[]):
                res.append({
                    "title":j.get("title"),"company":j.get("company",{}).get("display_name"),
                    "location":j.get("location",{}).get("display_name"),"url":j.get("redirect_url"),
                    "posted_at":j.get("created"),"salary":j.get("salary_min") or j.get("salary_max"),
                    "description":j.get("description","")
                })
        except: break
        time.sleep(0.35)
    df=pd.DataFrame(res)
    return filter_days(df,"posted_at")

def fetch_jooble():
    if not JOOBLE_KEY: return pd.DataFrame()
    host="br.jooble.org"
    res=[]
    for t in SEARCH_TERMS:
        body=json.dumps({"keywords":t,"location":"São Paulo","page":1})
        try:
            conn=http.client.HTTPConnection(host,timeout=30)
            conn.request("POST",f"/api/{JOOBLE_KEY}",body,{"Content-type":"application/json"})
            r=conn.getresponse()
            if r.status!=200: continue
            data=json.loads(r.read())
            for j in data.get("jobs",[]):
                res.append({
                    "title":j.get("title"),"company":j.get("company"),
                    "location":j.get("location"),"url":j.get("link"),
                    "posted_at":j.get("updated"),"salary":j.get("salary"),
                    "description":j.get("snippet","")
                })
        except: pass
        time.sleep(0.25)
    df=pd.DataFrame(res)
    return filter_days(df,"posted_at")

def run_pipeline():
    dfs=[fetch_remotive(), fetch_remoteok(), fetch_adzuna(), fetch_jooble()]
    dfs=[d for d in dfs if not d.empty]
    if not dfs: return pd.DataFrame()
    df=pd.concat(dfs,ignore_index=True)

    df["url_norm"]=df["url"].apply(normalize_url)
    df["posted_at"]=df["posted_at"].map(safe_to_dt)
    df["title"]=df["title"].astype(str).str.strip()
    df["company"]=df["company"].astype(str).str.strip()
    df["location"]=df["location"].astype(str).str.strip()

    df["mode"]=df.apply(mode_class,axis=1)

    df=df[
        df["title"].str.contains(TITLE_REGEX,na=False,regex=True) |
        df["description"].str.contains(TITLE_REGEX,na=False,regex=True)
    ]

    df=df[df.apply(lambda r: loc_ok(r["mode"],r["location"],r["company"],r["title"],r.get("description","")),axis=1)]

    df=df.drop_duplicates(subset=["url_norm"])
    df=df.drop_duplicates(subset=["company","title","location"])

    df=df[df["posted_at"]>= now_utc()-timedelta(days=DAYS)]

    df["matched_terms"]=df.apply(lambda r: terms_hit(r["title"]+" "+str(r.get("description",""))),axis=1)
    df["score"]=df.apply(score,axis=1)

    df=df.sort_values(["score","posted_at"],ascending=[False,False]).head(MAX_ROWS)
    return strip_tz(df)

📌 Cell 3 — Executar:

df = run_pipeline()
df.head(20)[["score","posted_at","title","company","location","mode","source","matched_terms","url"]] if not df.empty 

📌 Cell 4 — Exportar Excel

from datetime import datetime
from pathlib import Path
import pandas as pd

if df.empty:
    raise SystemExit("Nenhum dado para exportar")

df_x = df.copy()
df_x["posted_at"] = pd.to_datetime(df_x["posted_at"]).dt.strftime("%Y-%m-%d %H:%M")

df_x = df_x.sort_values(["score","posted_at"], ascending=[False,False]).reset_index(drop=True)

outfile = Path(f"vagas_{datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx")

with pd.ExcelWriter(outfile, engine="xlsxwriter", datetime_format="yyyy-mm-dd hh:mm") as writer:
    df_x.to_excel(writer, sheet_name="Consolidadas", index=False)

    for src in sorted(df_x["source"].unique()):
        df_x[df_x["source"] == src].to_excel(writer, sheet_name=src[:31], index=False)

print(f"✅ Arquivo gerado: {outfile.resolve()}")




