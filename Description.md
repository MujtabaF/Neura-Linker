# Neura-Linker (Student AI Matcher) — Technical Description

## Purpose

**Neura-Linker** is a **hybrid web system** that matches students for study groups or social pairing based on **survey-style profile fields**. It combines:

1. **Node.js (Express)** — Google Sheets as source of truth, static file hosting, email (Nodemailer), REST API for student rows
2. **Python (Flask)** — reads students from the Node API, runs **scikit-learn**-heavy matching (**KMeans**, encoders, optional hybrid similarity), exposes `/api/match`
3. **Browser UI** — `student-matcher.html`, `script.js`, `styles.css` for end users

Use this for **full-stack integration**, **ETL from Sheets**, **recommender systems**, and **Flask + Express polyglot** interviews.

---

## Architecture

```
Browser  →  Flask :5000  →  HTTP GET  →  Express :3001/api/students  →  Google Sheets API
                ↓
         matcher.generate_matches(DataFrame)
                ↓
         JSON matches (per student or filtered by ?name=)
```

**Critical operational detail:** Flask **`app.py`** calls `requests.get("http://localhost:3001/api/students")` with **retries** — if Node is down, matching returns **404** with explanatory JSON.

---

## Node server (`server.js`)

**Stack:** Express, **cors**, **dotenv**, **googleapis** (Sheets v4), **nodemailer**, **node-fetch** (ESM — `import` syntax).

**Static:** `express.static('.')` serves HTML/JS/CSS from project root.

### Google Sheets

- Env: `SPREADSHEET_ID`, `GOOGLE_CREDENTIALS` (JSON string of service account)
- **`initializeGoogleSheets`:** `GoogleAuth` with scope `spreadsheets`; builds `sheets` client
- **`ensureSheetAndHeaders`:** Ensures sheet tab **`Students`** exists; creates if missing
- **`createHeadersIfNeeded`:** If first row empty, writes headers:  
  `Name, Email, Major, Year, Language, Country, Personality, Study Style, Cuisine, Interests, Movies, Timestamp`

### Typical routes (file continues beyond excerpt)

- **GET `/api/students`** — returns rows as JSON for Flask matcher
- **POST** endpoints may accept form submissions from HTML and append rows
- **Email flows:** when a user registers or requests matches, may call Flask then send email via Nodemailer with match results

(Read full `server.js` for exact paths, validation, and error formatting helpers like `formatError`.)

**Port:** `process.env.PORT || 3001`

---

## Flask backend (`app.py`)

**Libraries:** Flask, **flask_cors**, **pandas**, **requests**, `matcher.generate_matches`, `config.MIN_SIMILARITY_SCORE`

### `load_students_data(max_retries=3, retry_delay=2)`

- GET Node API with timeout **15s**
- Handles **ConnectionError**, **Timeout**, **HTTPError** with retries and verbose logging
- Returns **list of dicts** or `[]`

### `GET /api/match`

1. Loads students; if empty → **404** with message about Node/Sheets
2. **`pd.DataFrame(students)`**
3. Resolves name column: `'name'` or `'Name'`
4. **`results = generate_matches(df)`** — dict keyed by **student name** → list of match objects
5. **Query param `name`:**  
   - Case-insensitive row lookup for student  
   - If not found → success with **empty matches** and friendly message (Sheets sync lag)  
   - Resolves match list with exact key then **case-insensitive** key fallback  
   - **Formats** each match for JSON: string coercion, `profile` dict flattened to JSON-safe types, `similarity_score` float, `commonalities` list of strings  
   - Returns `{ success, student_name, matches, total_matches }`  
6. **No `name` param:** returns **`all_matches`** dict and `total_students`

### `POST /api/match_for_student`

- JSON body `{ "name": "..." }` — loads all students, recomputes matches, returns slice for that student

### Run

- `app.run(host="0.0.0.0", port=5000, debug=True)` — **same port family as many Flask apps**; avoid conflict with Stone-See if both run locally.

---

## Matching engine (`matcher.py`)

**Large module** (2000+ lines) — core ideas:

### Similarity building blocks

- **`jaccard_index(set1, set2)`** — classic intersection/union; empty-empty → 1.0; one empty → 0.0
- **Multi-value fields** parsed from strings (lists / comma-separated) using `ast` or splits — configured in `config.REQUIRED_COLUMNS`, `MULTI_VALUE_FIELDS`, `CATEGORICAL_FIELDS`

### Diversity

- **`calculate_diversity_bonus`**, **`is_diverse_match`** — if `ENABLE_DIVERSITY_ENFORCEMENT`, adds capped bonus when students differ on attributes like country, major (see `DIVERSITY_ATTRIBUTES`, `DIVERSITY_BONUS` in `config.py`)

### Clustering path (`config.MATCHING_ALGORITHM`)

- **`kmeans`** (default in config excerpt):  
  - Feature pipeline: **StandardScaler**, **LabelEncoder** / **MultiLabelBinarizer** as appropriate  
  - **KMeans** with **auto cluster count** between `KMEANS_MIN_CLUSTERS` and `KMEANS_MAX_CLUSTERS` (elbow / silhouette — see implementation)  
  - Limits **`MAX_MATCHES_FROM_SAME_CLUSTER`**; optional **`ENABLE_CROSS_CLUSTER_MATCHING`**

### Weighted similarity

- **`MATCHING_WEIGHTS`** in `config.py` (sum 100): major 30, personality 25, interests 22 (Jaccard), year 6, studyStyle 7, country 5, language 1, cuisine 2, movies 2  
- **Thresholds:** `SIMILARITY_THRESHOLDS` for qualitative bands  
- **`TOP_MATCHES_COUNT`** default 3  
- **`MIN_SIMILARITY_SCORE`** base 25 with **dynamic adjustment** by cohort size (documented in config comments)

### Outputs per match

- Typically: `name`, `email`, `similarity_score`, `explanation` (natural language or template), `commonalities`, `profile` snapshot

### Evaluation tooling

- **`comprehensive_evaluation.py`** — offline metrics for tuning weights/thresholds (read for precision@k, coverage, etc.)

---

## Configuration (`config.py`)

Centralizes **all** tuning: weights, thresholds, algorithm mode, KMeans bounds, diversity, tie-breaking (`ENABLE_TIE_BREAKING`, `RANDOM_SEED`), precision decimals, debug logging, required columns, fallback weights.

**Interview tip:** Emphasize **business rules as data** — non-engineers could tune `config.py` with guidance.

---

## Frontend

- **`student-matcher.html`** — form or flow to collect user identity and display matches
- **`script.js`** — calls Flask endpoints, renders match cards
- **`styles.css`** — layout

---

## Developer experience

- **`start-servers.sh` / `start-servers.bat`** — launch Node and/or Flask in correct order
- **`requirements.txt`** — Python deps (pandas, sklearn, flask, requests, …)
- **`package.json`** — Node deps (express, googleapis, nodemailer, …)
- **`INSTALLATION.md`** — step-by-step env and credentials

---

## Security and privacy

- Service account JSON in **environment** — never commit
- Student **PII** (email) in API responses — use **HTTPS** and **auth** in production
- Flask **debug=True** insecure for public hosts

---

## How to summarize by role

- **Backend (Node):** Google Sheets CRUD, service account auth, REST facade for Python
- **Backend (Python):** Pandas feature matrix, KMeans + weighted similarity, JSON API
- **ML / recommender:** Jaccard for multi-select fields, clustering to reduce search space, diversity bonuses
- **Full-stack:** Two-process architecture, retry logic, case-insensitive name resolution
