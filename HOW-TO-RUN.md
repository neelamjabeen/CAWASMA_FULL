# CAWASMA — Complete Run & Usage Guide
### (Context-Aware Web & API Security Misconfiguration Analyzer)

Ye guide aap ko qadam-ba-qadam batata hai ke CAWASMA tool ko **kaise chalana hai**
aur **kaise use karna hai** — chahe aap Docker use karein ya na karein. Commands
copy-paste karne ke liye tayyar hain.

> **Tip:** Agar aap pehli dafa chala rahe hain, to **Tareeqa 1 (Docker)** use karein.
> Ek command se poora project (database + redis + backend + worker + frontend) uth jata hai.

---

## 0. Project ka structure (ek nazar mein)

```
cawasma/
├── app/              ← FastAPI backend (API, scanner, database, services)
├── celery_worker/    ← background worker (scans yahan chalte hain)
├── migrations/       ← database tables banane ki migrations (Alembic)
├── tests/            ← 88 automated tests
├── frontend/         ← React (Vite) dashboard — UI
├── docker-compose.yml← saare 5 services ek saath
├── Dockerfile        ← backend image
├── requirements.txt  ← backend ke Python packages
├── .env.example      ← settings ka namoona (ise copy kar ke .env banayenge)
└── README.md
```

---

## 1. Kya kya chahiye (Prerequisites)

**Tareeqa 1 (Docker) ke liye — sirf 2 cheezein:**
- **Docker Desktop** (Windows/Mac) — WSL2 integration ON honi chahiye
- Bas. Python/Node khud container ke andar aa jate hain.

**Tareeqa 2/3 (bina Docker) ke liye:**
- **Python 3.11+**
- **Node.js 18+** (frontend ke liye)
- (optional) PostgreSQL + Redis — ya in ke baghair SQLite + inline mode (neeche samjhaya hai)

---

## 2. TAREEQA 1 — Docker se (SABSE AASAAN, RECOMMENDED) ⭐

Ye poora project ek command se chalata hai.

### Qadam 1 — Project kholein
```bash
cd cawasma          # jahan docker-compose.yml hai
```

### Qadam 2 — Settings file banayein
```bash
cp .env.example .env
```
(Default values theek hain — kuch change karne ki zaroorat nahi.)

### Qadam 3 — Sab kuch chalayein
```bash
docker compose up --build
```
Pehli dafa thoda time lagega (images download + build honge). Jab ready ho jaye to
terminal mein `api`, `worker`, `frontend`, `db`, `redis` sab chalte dikhenge.

### Qadam 4 — Browser mein kholein
| Cheez            | Address                        |
|------------------|--------------------------------|
| **Frontend (UI)**| http://localhost:5173          |
| Backend API docs | http://localhost:8000/docs     |
| API health check | http://localhost:8000/api/health |

Bas! Ab seedha **"Tareeqa 4 — Tool kaise use karein"** par chale jayein.

### Band karne ke liye:
Terminal mein `Ctrl + C`, phir (data bhi delete karna ho to):
```bash
docker compose down            # services band
docker compose down -v         # services + database data delete
```

---

## 3. TAREEQA 2 — Bina Docker (manual)

Agar Docker nahi hai, to backend aur frontend alag alag chalayenge.
**Do sub-options hain** — pehla (SQLite + inline) sabse halka hai, laptop/viva ke liye best.

### 3-A. Backend chalana

**Terminal 1 — Backend:**
```bash
cd cawasma
python -m venv .venv

# virtual environment activate karein:
source .venv/bin/activate        # Linux / Mac / WSL
# .venv\Scripts\activate         # Windows PowerShell

pip install -r requirements.txt
```

Ab do raste hain:

#### Option A — Sabse halka: SQLite + inline (koi Postgres/Redis/Celery nahi) ✅
```bash
pip install aiosqlite                               # SQLite driver

# settings (Linux/Mac/WSL):
export DATABASE_URL="sqlite+aiosqlite:///./cawasma.db"
export ALEMBIC_SQLALCHEMY_URL="sqlite:///./cawasma.db"
export RUN_SCANS_INLINE=true                        # scans backend ke andar hi chalenge

alembic upgrade head                                # database tables banayein
uvicorn app.main:app --reload                       # backend start
```
> Windows PowerShell mein `export X=Y` ki jagah `$env:X="Y"` likhein.

#### Option B — Production jaisa: PostgreSQL + Redis + Celery
```bash
# .env mein DATABASE_URL / REDIS_URL apne Postgres/Redis ke mutabiq set karein
cp .env.example .env
alembic upgrade head
uvicorn app.main:app --reload       # Terminal 1

# Terminal 2 — Celery worker (alag terminal, venv activate kar ke):
celery -A celery_worker.celery_app.celery_app worker --loglevel=info
```

Backend ab chal raha hai: **http://localhost:8000** (docs: `/docs`).

### 3-B. Frontend chalana

**Terminal 2 (ya 3) — Frontend:**
```bash
cd cawasma/frontend
npm install
npm run dev
```
Frontend khul jayega: **http://localhost:5173**

> Frontend by default `http://localhost:8000` par backend dhoondta hai. Agar backend
> kisi aur address par hai to `frontend/.env` mein `VITE_API_BASE=...` set karein.

---

## 4. TOOL KAISE USE KAREIN (chalne ke baad) 🖥️

1. Browser mein **http://localhost:5173** kholein → **Dashboard** aayega.
2. Upar/side mein **"New Scan"** par click karein.
3. Target URL likhein — **HTTPS zaroori hai**, misal:
   ```
   https://example.com
   ```
   (Try buttons bhi diye hain: example.com, scanme.nmap.org, badssl.com)
4. **"Launch Scan"** dabayein.
5. Scan page khul jayega jahan **live progress bar** chalega (crawling → checks →
   chains). Ye khud update hoti rehti hai (WebSocket se).
6. Scan `COMPLETED` hone par aap ko milega:
   - **Stat cards** — Endpoints, Findings, Attack Chains, Highest Score
   - **Severity Distribution** — Critical/High/Medium/Low ki bar
   - **Findings tab** — har finding ka severity + **context-aware score**.
     Kisi finding par click karein → detail drawer khulega (CVSS base × context +
     bonus = final score, evidence redacted, recommendation, limitation).
   - **Attack Chains tab** — findings ka combination jo bara khatra banata hai.
   - **Endpoints tab** — mile hue endpoints + un ki context class.
7. Upar daayein **Report** buttons se report download karein: **HTML / PDF / CSV / JSON**.

> **Important (viva ke liye):** CAWASMA ki novelty yahi hai — **same finding alag
> endpoint par alag score paata hai**. Misal: "Missing X-Frame-Options" `/about`
> (LOW) par ~3.7, lekin `/api/payment/checkout` (CRITICAL) par 10.0. Ye detail
> drawer mein saaf nazar aata hai.

---

## 5. Tests chalana (sab kuch sahi hai — sabit karne ke liye)

```bash
cd cawasma
source .venv/bin/activate
pip install pytest pytest-asyncio aiosqlite greenlet   # agar pehle se nahi
python -m pytest -q
```
Expected: **88 passed** ✅

---

## 6. Optional features (on karna ho to)

### (a) NLP embedding layer (sentence-transformers)
Classifier bina model ke bhi chalta hai (keyword layer). Semantic layer on karne ke liye:
```bash
pip install -r requirements-ml.txt     # ~200MB, ek dafa
```
Na karein to bhi tool poori tarah chalega — demo kabhi nahi tootega.

### (b) LLM (URL abbreviation expansion — usr→user wagera)
`.env` mein:
```
LLM_ENABLED=true
LLM_PROVIDER=anthropic          # ya openai
ANTHROPIC_API_KEY=sk-...        # apni key (ya OPENAI_API_KEY)
```
Key na ho to automatically built-in dictionary use hoti hai.

---

## 7. Masle aur hal (Troubleshooting) 🔧

### Docker mein scan slow hai ya "chal hi nahi raha" — sabse pehle ye

Scan ab **parallel** chalta hai (ek saath 10 endpoints), to pehle se kaafi tez hai.
Phir bhi masla ho to tarteeb se ye karein:

1. **Worker ke logs dekhein** (sabse zaroori) — alag terminal mein:
   ```bash
   docker compose logs -f worker
   ```
   Scan start karne par yahan `worker received scan ...` aana chahiye. Agar nahi
   aata → task dispatch nahi ho raha. Agar error aata hai → wahi asli masla hai.

2. **Worker ka internet check karein** (scan target tak pahunch):
   ```bash
   docker compose exec worker python -c "import httpx; print(httpx.get('https://example.com', timeout=10).status_code)"
   ```
   `200` aaye to network theek. Agar timeout/error → container ko internet nahi mil
   raha (ye "scan nothing" ki sabse badi wajah hai) → Docker Desktop restart, ya
   WSL DNS fix (`/etc/docker/daemon.json` mein `{"dns":["8.8.8.8","1.1.1.1"]}`).

3. **Scan QUEUED par atka hai?** Matlab worker task nahi utha raha:
   ```bash
   docker compose ps          # worker "Up" hai?
   docker compose restart worker
   ```

4. **Jaldi test ke liye chhota target lein** — pehli dafa `https://example.com`
   use karein (halka site), `scanme.nmap.org` ya bara site baad mein.

> **Tip:** Agar Docker tang kar raha ho to demo ke liye **bina-Docker inline mode**
> (Section 4, Option A) sabse reliable hai — Celery/Redis ki zaroorat hi nahi,
> scan seedha backend ke andar chalta hai.

### Aam masle

| Masla | Hal |
|------|-----|
| **UI par peela banner "Backend not reachable, demo data"** | Backend nahi chal raha. Dekhein `http://localhost:8000/api/health` khulta hai ya nahi. Backend start karein. |
| **`docker compose` DNS / download error (WSL)** | `/etc/docker/daemon.json` banayein: `{"dns":["8.8.8.8","1.1.1.1"]}` phir Docker restart. |
| **Port already in use (8000 ya 5173)** | Us port par chalta purana process band karein, ya compose/uvicorn/vite mein doosra port set karein. |
| **`alembic upgrade head` error** | Sahi folder (`cawasma/`) mein hain? `DATABASE_URL` set hai? SQLite path mein `ALEMBIC_SQLALCHEMY_URL` bhi set karein. |
| **PDF report 501 deta hai** | WeasyPrint ki native libraries chahiyein — Docker image mein already hain. Bina-Docker local par PDF ke liye system libs install karni parti hain; HTML/CSV/JSON hamesha chalti hain. |
| **Scan `QUEUED` par atak gaya (bina Docker)** | Ya to Celery worker chalayein (Option B), ya `RUN_SCANS_INLINE=true` set karein (Option A). |
| **CORS error browser console mein** | `.env` mein `FRONTEND_ORIGIN` wahi hona chahiye jahan frontend chal raha hai (default `http://localhost:5173`). |

---

## 8. Quick Reference (yaad-dasht)

```bash
# --- Docker (sab kuch) ---
cp .env.example .env
docker compose up --build        # start
docker compose down              # stop

# --- Bina Docker: backend (halka) ---
source .venv/bin/activate
export DATABASE_URL="sqlite+aiosqlite:///./cawasma.db"
export ALEMBIC_SQLALCHEMY_URL="sqlite:///./cawasma.db"
export RUN_SCANS_INLINE=true
alembic upgrade head
uvicorn app.main:app --reload

# --- Bina Docker: frontend ---
cd frontend && npm install && npm run dev

# --- Tests ---
python -m pytest -q
```

| Address | Kya hai |
|---------|---------|
| http://localhost:5173 | Frontend (UI) — yahan kaam karein |
| http://localhost:8000/docs | Swagger API docs |
| http://localhost:8000/api/health | Backend zinda hai ya nahi |

---

## 9. Scan ka safar (andar kya hota hai)

```
Target URL  →  Validation + SSRF check  →  Endpoint discovery (crawler)
   →  Har endpoint ki context classification (NLP)
   →  58 security checks  →  Response body classification
   →  CVSS base × context multiplier + response bonus = FINAL score
   →  Attack-chain correlation  →  Database  →  Report + live UI
```

---

**Mubarak ho — CAWASMA ab chalne ke liye tayyar hai.** Kisi bhi qadam par masla ho to
us step ka error message save kar lein; `HOW-TO-RUN.md` ka Troubleshooting section
aksar jawab de deta hai.
