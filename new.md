To make a word cloud like the one you uploaded — **per state** — showing which words people mention the most in Reddit posts, follow this step-by-step using your `df` (which includes a `state` and `cleaned_text` column):

---

## ✅ Step-by-Step: Generate a Word Cloud Per State

### ✅ 1. Install the wordcloud library

If not already installed:

```bash
pip install wordcloud
```

---

### ✅ 2. Import and Set Up

```python
from wordcloud import WordCloud
import matplotlib.pyplot as plt
```

---

### ✅ 3. Create and Plot a Word Cloud for One State

```python
def plot_wordcloud(state_name, df):
    text = " ".join(df[df['state'] == state_name]['cleaned_text'].dropna())
    wordcloud = WordCloud(width=800, height=400, background_color='white', colormap='inferno').generate(text)

    plt.figure(figsize=(10, 5))
    plt.imshow(wordcloud, interpolation='bilinear')
    plt.axis('off')
    plt.title(f"Most Common Words in {state_name}", fontsize=16)
    plt.show()

# Example
plot_wordcloud("Arizona", df)
```

---

### ✅ 4. Automatically Generate One for Each State

```python
for state in df['state'].unique():
    plot_wordcloud(state, df)
```

---

### 🧠 Optional Enhancements

* Filter by **issue** (e.g., only posts labeled as “Economy”)
* Add stopwords (like "people", "thing") with `stopwords` argument
* Export each image with `plt.savefig(f"{state}_wordcloud.png")`

---

Would you like help filtering these by `top_issue` or saving them to disk as image files for a report?
