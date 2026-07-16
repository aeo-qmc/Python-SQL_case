## 04 · Incident classification with LLM `Python`

**Context:** this script automatically classifies network incidents that have no category yet, using an LLM API. It runs every night as a scheduled routine.

**What do you notice about this code?**

```python
import openai
import psycopg2
import os
from dotenv import load_dotenv

load_dotenv()

openai.api_key = "sk-proj-aBcDeFgHiJkLmNoPqRsTuVwX..."

def classify_incidents():
    conn = psycopg2.connect(
        host=os.getenv("DB_HOST"),
        dbname=os.getenv("DB_NAME"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
    )
    cur = conn.cursor()

    cur.execute("SELECT id, description FROM incidents WHERE category IS NULL")
    incidents = cur.fetchall()

    for incident_id, description in incidents:
        response = openai.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "user", "content": f"Classify this network incident into a category: {description}"}
            ]
        )
        category = response.choices[0].message.content

        cur.execute(f"UPDATE incidents SET category = '{category}' WHERE id = {incident_id}")
        conn.commit()

    cur.close()
    conn.close()

if __name__ == "__main__":
    classify_incidents()
```

---
