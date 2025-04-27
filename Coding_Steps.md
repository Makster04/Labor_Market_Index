Below is a *reference implementation* that maps **every roadmap phase to concrete, runnable Python (or Streamlit/FastAPI) code**.  
All snippets assume a repo structured like this:

```
nlp-swing-state/
├─ data/                # raw → bronze → silver
├─ notebooks/           # exploratory, 1-off validation
├─ src/
│  ├─ config.py
│  ├─ ingest/
│  │   ├─ reddit.py
│  │   ├─ twitter.py
│  │   └─ youtube.py
│  ├─ preprocess/
│  │   └─ cleaner.py
│  ├─ nlp/
│  │   ├─ topic.py
│  │   ├─ sentiment.py
│  │   └─ cluster.py
│  ├─ trends/
│  │   └─ detector.py
│  ├─ dashboard/
│  │   └─ app.py
│  └─ api/
│      └─ main.py
└─ requirements.txt
```

---

## 0  Project Kick-Off & Config

```python
# src/config.py
SWING_STATES = ["WI", "PA", "MI", "AZ", "GA", "NV", "NC"]

PATHS = dict(
    RAW="data/raw/",
    BRONZE="data/bronze/",
    SILVER="data/silver/",
    MODELS="data/models/",
)

KPI = {
    "TOPIC_COHERENCE_TARGET": 0.75,
    "DASH_LATENCY_HR":        4
}
```

---

## 1  Data Sourcing “Firehose”

### 1.1  Reddit (Pushshift)

```python
# src/ingest/reddit.py
import requests, json, gzip, os, datetime as dt
from pathlib import Path
from src.config import PATHS, SWING_STATES

def fetch_pushshift(state_abbrev: str, after: str, before: str, size=500) -> list[dict]:
    query = f"subreddit:r/{state_abbrev.lower()}*"
    url   = (
        f"https://api.pushshift.io/reddit/comment/search?"
        f"q=&{query}&after={after}&before={before}&size={size}&lang=en"
    )
    return requests.get(url, timeout=30).json()["data"]

def daily_job(target_date: dt.date):
    after  = int(dt.datetime.combine(target_date, dt.time.min).timestamp())
    before = int(dt.datetime.combine(target_date, dt.time.max).timestamp())

    out_dir = Path(PATHS["RAW"]) / "reddit" / target_date.isoformat()
    out_dir.mkdir(parents=True, exist_ok=True)

    for st in SWING_STATES:
        data = fetch_pushshift(st, after, before)
        fn   = out_dir / f"{st}.jsonl.gz"
        with gzip.open(fn, "wt") as fp:
            for row in data:
                json.dump(row, fp)
                fp.write("\n")
```

> Schedule `daily_job()` via Airflow or GitHub Actions (cron).

### 1.2  Twitter/X *(tweepy v2)*

```python
# src/ingest/twitter.py
import tweepy, os, gzip, json
from datetime import datetime, timedelta
from src.config import PATHS, SWING_STATES

client = tweepy.Client(os.getenv("TW_BEARER"))

def search(state, start_time, end_time, max_pages=100):
    query = f"place_country:US lang:en -is:retweet ({state})"
    paginator = tweepy.Paginator(
        client.search_recent_tweets,
        query=query,
        tweet_fields=["author_id","created_at","public_metrics","text"],
        max_results=100,
        start_time=start_time,
        end_time=end_time,
    )
    for page in paginator.take(max_pages):
        for t in page.data or []:
            yield t.data

def collect_day(day: datetime):
    start, end = day.isoformat()+"Z", (day+timedelta(days=1)).isoformat()+"Z"
    path = f"{PATHS['RAW']}/twitter/{day.date()}"
    os.makedirs(path, exist_ok=True)
    for st in SWING_STATES:
        fn = f"{path}/{st}.jsonl.gz"
        with gzip.open(fn, "wt") as fp:
            for tw in search(st, start, end):
                json.dump(tw, fp); fp.write("\n")
```

*Add similar modules for YouTube comments or RSS.*

---

## 2  Text Pre-Processing Pipeline

```python
# src/preprocess/cleaner.py
import re, json, spacy, emoji, gzip, pathlib
from tqdm import tqdm
from src.config import PATHS, SWING_STATES

nlp = spacy.load("en_core_web_sm", disable=["ner","parser","tagger"])

EMOJI_RE = emoji.get_emoji_regexp()
URL_RE   = re.compile(r"https?://\S+")

def normalize(text:str) -> str:
    text = text.lower()
    text = EMOJI_RE.sub(lambda m: f" {emoji.demojize(m.group())} ", text)
    text = URL_RE.sub(" ", text)
    text = re.sub(r"\s+", " ", text).strip()
    return text

def spacy_pipe(texts):
    for doc in nlp.pipe(texts, batch_size=1000):
        yield " ".join(t.lemma_ for t in doc if not t.is_stop and t.is_alpha)

def process_file(fn_in: pathlib.Path, fn_out: pathlib.Path):
    with gzip.open(fn_in, "rt") as src, gzip.open(fn_out, "wt") as tgt:
        raw  = [json.loads(l)["body" if "reddit" in fn_in.parts else "text"] for l in src]
        clean= spacy_pipe(map(normalize, raw))
        for c in clean:
            json.dump({"clean": c}, tgt)
            tgt.write("\n")
```

> **Run** `process_file()` across yesterday’s raw dumps → **Bronze**→**Silver** folders.

---

## 3  Core NLP Modeling (Topics + Sentiment)

```python
# src/nlp/topic.py
from bertopic import BERTopic
from sklearn.feature_extraction.text import CountVectorizer
import joblib, pathlib
from src.config import PATHS

def train_bertopic(texts:list[str]):
    vectorizer = CountVectorizer(min_df=10, ngram_range=(1,2))
    model = BERTopic(language="english", vectorizer_model=vectorizer,
                     calculate_probabilities=True)
    topics, _ = model.fit_transform(texts)
    model.save(pathlib.Path(PATHS["MODELS"])/"bertopic")
    return model, topics

# --- Sentiment
# src/nlp/sentiment.py
from nltk.sentiment import SentimentIntensityAnalyzer
import pandas as pd, nltk, pathlib, json, gzip
nltk.download("vader_lexicon")

def annotate_sentiment(fn_in, fn_out):
    sia = SentimentIntensityAnalyzer()
    with gzip.open(fn_in,"rt") as f, gzip.open(fn_out,"wt") as g:
        for line in f:
            t  = json.loads(line); txt = t["clean"]
            sc = sia.polarity_scores(txt)
            t.update(sc)
            g.write(json.dumps(t)+"\n")
```

---

## 4  Segment Clustering & SHAP-Style Explanations

```python
# src/nlp/cluster.py
from sentence_transformers import SentenceTransformer
from sklearn.cluster import KMeans
from sklearn.decomposition import PCA
from sklearn.metrics import silhouette_score
import pandas as pd, joblib
import shap, numpy as np

def embed_texts(texts):
    model = SentenceTransformer("all-MiniLM-L6-v2")
    return model.encode(texts, show_progress_bar=True)

def best_k(X, k_min=3, k_max=12):
    best, best_k = -1, k_min
    for k in range(k_min, k_max+1):
        score = silhouette_score(X, KMeans(k).fit_predict(X))
        if score > best: best, best_k = score, k
    return best_k

def cluster(texts):
    X = embed_texts(texts)
    k = best_k(X)
    km= KMeans(k, n_init="auto").fit(X)
    joblib.dump(km, f"{PATHS['MODELS']}/kmeans.joblib")
    return km.labels_

# --- keyword explanations
def shap_keywords(texts, labels, km):
    explainer = shap.KMeans(km)
    shap_vals = explainer(texts)          # array (n,k)
    keywords  = {}
    for lbl in np.unique(labels):
        idx    = np.where(labels==lbl)
        top_i  = np.argsort(-shap_vals[idx][:,lbl])[:20]
        keywords[lbl] = [texts[i] for i in top_i]
    return keywords
```

---

## 5  Trend & Alert Engine

```python
# src/trends/detector.py
import pandas as pd, numpy as np

def daily_topic_share(df:pd.DataFrame, state:str):
    """df cols: date, state, topic_id, prob"""
    daily = (df[df.state==state]
             .groupby(["date","topic_id"])
             .prob.mean()
             .unstack(fill_value=0))
    return daily

def burst_flags(series:pd.Series, lookback=14, z_thresh=2):
    mu = series.rolling(lookback).mean()
    sd = series.rolling(lookback).std()
    z  = (series - mu) / sd
    return (z > z_thresh)
```

Save bursts to a Timescale table (`state, topic_id, date, is_burst`).

---

## 6  Streamlit Dashboard (MVP)

```python
# src/dashboard/app.py
import streamlit as st, pandas as pd, plotly.express as px
from sqlalchemy import create_engine
engine = create_engine("postgresql://...")

st.title("🗳️ Swing-State Voter Insight Engine")

state = st.selectbox("Choose a swing state", ["WI","PA","MI","AZ","GA","NV","NC"])
topic_map = pd.read_sql("select * from topics", engine)

# Heat-map -------------------------------------------------------------
st.subheader("Topic Prevalence Over Time")
df = pd.read_sql(f"""
    select date, topic_id, prevalence
    from topic_daily
    where state='{state}'
    order by date
""", engine)
fig = px.imshow(df.pivot("topic_id","date","prevalence"), aspect="auto")
st.plotly_chart(fig, use_container_width=True)

# Sentiment gauge ------------------------------------------------------
st.subheader("Overall Sentiment (last 7 days)")
sent = pd.read_sql(f"""
   select avg(compound) as cmp
   from sentiment
   where state='{state}' and date >= current_date - 7
""", engine).iloc[0,0]
st.progress((sent+1)/2)  # map -1…1 → 0…1
```

Deploy with `streamlit run src/dashboard/app.py`.

---

## 7  Validation Notebook (Excerpt)

```python
# notebooks/07_validation.ipynb
import pandas as pd, matplotlib.pyplot as plt, seaborn as sns
from gensim.models.coherencemodel import CoherenceModel
from bertopic import BERTopic, utils

model = BERTopic.load("data/models/bertopic")
docs  = pd.read_parquet("data/silver/clean.parquet")["clean"].tolist()

cm = CoherenceModel(
    topics        = model.get_topics(),
    texts         = [d.split() for d in docs],
    dictionary    = utils.words_to_token_ids(docs),
    coherence     = 'c_v'
)
print("Topic coherence:", cm.get_coherence())   # target ≥0.75
```

---

## 8  Monetization API (FastAPI)

```python
# src/api/main.py
from fastapi import FastAPI, Query
import pandas as pd, sqlalchemy as sa

app = FastAPI(title="Swing-State Insight API")
engine = sa.create_engine("postgresql://...")

@app.get("/trends")
def get_trends(state:str=Query(...), topic_id:int=Query(...)):
    sql = """
        select date, prevalence
        from topic_daily
        where state=:st and topic_id=:tid
        order by date desc limit 30
    """
    df = pd.read_sql(sql, engine, params={"st":state,"tid":topic_id})
    return df.to_dict(orient="records")

@app.get("/segments/{cluster_id}")
def segment_keywords(cluster_id:int):
    sql = "select keyword, weight from cluster_keywords where cluster=:cid"
    return pd.read_sql(sql, engine, params={"cid":cluster_id}).to_dict("records")
```

Run:

```bash
uvicorn src.api.main:app --reload --port 8080
```

---

### How to **Run End-to-End** on a 1-Day Sample

```bash
git clone <repo>; cd nlp-swing-state
pip install -r requirements.txt

# 1. ingest
python -m src.ingest.reddit   --day 2025-04-25
python -m src.ingest.twitter  --day 2025-04-25

# 2. preprocess
python -m src.preprocess.cleaner  data/raw/...  data/bronze/...

# 3. model
python - <<'PY'
from pathlib import Path, glob
from src.nlp.topic import train_bertopic
texts = [l["clean"] for f in glob.glob("data/bronze/**/*.gz")
                     for l in map(json.loads, gzip.open(f))]
train_bertopic(texts)
PY

# 4. dashboard
streamlit run src/dashboard/app.py
```

---

## What’s Next?

- **Need a quick-start Colab?** I can generate a notebook that walks through the first three phases on a 10 MB sample.  
- **Prefer the GitHub skeleton pre-made?** I can push the directory tree with stub files and CI YAML.  
- **Want visual goodies?** Happy to output a SHAP keyword bar-plot or a Streamlit GIF.

Tell me which artifact you’d like created first and I’ll spin it up!