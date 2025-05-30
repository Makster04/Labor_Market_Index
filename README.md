Great! Here’s a **step-by-step guide** to:

1. ✅ Set up Reddit API credentials
2. ✅ Use `praw` to fetch **both posts and comments**
3. ✅ Preprocess and run **basic sentiment analysis**

---

## 🔐 STEP 1: Set Up Reddit API Credentials

1. Go to [https://www.reddit.com/prefs/apps](https://www.reddit.com/prefs/apps)
2. Scroll down and click: **"create app"**
3. Fill out:

   * **Name:** `swing_state_scraper`
   * **Type:** `script`
   * **Redirect URI:** `http://localhost`
4. After submission, you’ll get:

   * `client_id` (below the app name)
   * `client_secret` (in the app details)

---

## 📦 STEP 2: Install `praw`

```bash
pip install praw
```

---

## 🧪 STEP 3: Use `praw` to Fetch Posts + Comments

```python
import praw
import pandas as pd
from datetime import datetime

# 🔑 Replace with your actual credentials
reddit = praw.Reddit(
    client_id="YOUR_CLIENT_ID",
    client_secret="YOUR_CLIENT_SECRET",
    user_agent="swing_state_scraper"
)

def fetch_reddit_posts_and_comments(subreddit_name, limit=50):
    subreddit = reddit.subreddit(subreddit_name)
    data = []

    for submission in subreddit.hot(limit=limit):
        if not submission.stickied:
            post = {
                'title': submission.title,
                'text': submission.selftext,
                'created_utc': datetime.utcfromtimestamp(submission.created_utc),
                'subreddit': subreddit_name,
                'comments': []
            }

            submission.comments.replace_more(limit=0)  # Load all comments
            post['comments'] = [comment.body for comment in submission.comments[:10]]  # Top 10 comments

            data.append(post)

    return pd.DataFrame(data)

# Example usage
df = fetch_reddit_posts_and_comments('PApolitics', limit=20)
print(df.head(3))
```

---

## 💬 STEP 4: Sentiment Analysis with TextBlob

```bash
pip install textblob
python -m textblob.download_corpora
```

```python
from textblob import TextBlob

def analyze_sentiment(text):
    if not text:
        return 0
    return TextBlob(text).sentiment.polarity  # [-1 to 1]

# Add sentiment scores
df['post_sentiment'] = df['text'].apply(analyze_sentiment)
df['avg_comment_sentiment'] = df['comments'].apply(lambda comments: 
    sum(analyze_sentiment(c) for c in comments) / len(comments) if comments else 0
)

print(df[['title', 'post_sentiment', 'avg_comment_sentiment']].head())
```

---

## ✅ Optional: Filter for Swing State Topics

You can apply keyword filters like:

```python
keywords = ['healthcare', 'jobs', 'abortion', 'biden', 'trump', 'inflation']
df_filtered = df[df['text'].str.contains('|'.join(keywords), case=False)]
```

---

Would you like to:

* Turn this into a live dashboard or report?
* Extract topics (LDA or BERTopic)?
* Expand to multiple swing-state subreddits (e.g. `r/WisconsinPolitics`, `r/Michigan`)?

Let me know how far you’d like to scale this.
