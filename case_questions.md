# Interview Case

---

## 01 · Active contract filtering `SQL`

### Schema

```
contracts(
  contract_id  INT,
  site_id      INT,
  operator     TEXT,
  base_fee     NUMERIC,   -- monthly fee in R$
  status       TEXT       -- 'active' | 'churned'
)
```

### Sample data

| contract_id | site_id | operator | base_fee | status  |
|-------------|---------|----------|----------|---------|
| 1           | 101     | Claro    | 8,200    | active  |
| 2           | 101     | TIM      | 4,500    | active  |
| 3           | 102     | Claro    | 6,100    | active  |
| 4           | 102     | Vivo     | 5,300    | active  |
| 5           | 103     | TIM      | 3,800    | active  |
| 6           | 103     | TIM      | 4,200    | active  |
| 7           | 103     | TIM      | 5,900    | active  |
| 8           | 104     | Vivo     | 9,100    | active  |
| 9           | 104     | Claro    | 2,100    | churned |
| 10          | 105     | Claro    | 7,400    | active  |

### Question

Find operators whose **active** contracts have an **average base fee above 5,000** — but only consider operators who have **at least 3 active contracts**. Return the operator name and their average base fee, sorted descending.

**Which of the queries below correctly solves this? Are any wrong? Explain each.**

#### Option A

```sql
SELECT operator, AVG(base_fee) AS avg_fee
FROM contracts
WHERE status = 'active' AND AVG(base_fee) > 5000
GROUP BY operator
HAVING COUNT(*) >= 3
ORDER BY avg_fee DESC;
```

#### Option B

```sql
SELECT operator, AVG(base_fee) AS avg_fee
FROM contracts
WHERE status = 'active'
GROUP BY operator
HAVING COUNT(*) >= 3 AND AVG(base_fee) > 5000
ORDER BY avg_fee DESC;
```

#### Option C

```sql
SELECT operator, AVG(base_fee) AS avg_fee
FROM contracts
WHERE status = 'active' AND base_fee > 5000
GROUP BY operator
HAVING COUNT(*) >= 3
ORDER BY avg_fee DESC;
```

---

## 02 · External API integration routine `Python`

**Context:** the code below was written to fetch site data from an external API and save it to PostgreSQL. It works in some cases, but we want to ship it as a daily production routine.

**What problems do you see? What would you improve?**

```python
import requests
import psycopg2
from datetime import datetime

API_URL = "https://api.exemplo.com/sites"
DB_CONN = "host=localhost dbname=dw user=admin password=admin123"

def run():
    response = requests.get(API_URL)
    data = response.json()

    conn = psycopg2.connect(DB_CONN)
    cursor = conn.cursor()

    for site in data:
        site_id = site["id"]
        name = site["name"]
        operator = site["operator"]
        lat = site["lat"]
        lon = site["lon"]

        cursor.execute(
            f"""
            INSERT INTO telecom_sites (
                site_id, name, operator, latitude, longitude, created_at
            )
            VALUES (
                {site_id}, '{name}', '{operator}', {lat}, {lon}, '{datetime.now()}'
            )
            """
        )

    conn.commit()
    cursor.close()
    conn.close()

if __name__ == "__main__":
    run()
```

---

## 03 · Revenue and incident report `SQL`

### Schema

```
sites(
  site_id  INT,
  name     TEXT
)

contracts(
  contract_id  INT,
  site_id      INT,
  operator     TEXT,
  base_fee     NUMERIC,  -- monthly fee in R$
  status       TEXT      -- 'active' | 'churned'
)

incidents(
  incident_id  INT,
  site_id      INT,
  severity     TEXT      -- 'critical' | 'high' | 'low'
)
```

### Sample data

**sites**

| site_id | name        |
|---------|-------------|
| 101     | Alpha Tower |
| 102     | Beta Mast   |
| 103     | Gamma Site  |

**contracts**

| contract_id | site_id | operator | base_fee | status |
|-------------|---------|----------|----------|--------|
| 1           | 101     | Claro    | 8,200    | active |
| 2           | 101     | TIM      | 4,500    | active |
| 3           | 102     | Vivo     | 6,100    | active |
| 4           | 103     | Claro    | 3,800    | active |
| 5           | 103     | TIM      | 5,200    | active |

**incidents**

| incident_id | site_id | severity |
|-------------|---------|----------|
| 1           | 101     | critical |
| 2           | 101     | high     |
| 3           | 101     | low      |
| 4           | 102     | critical |
| 5           | 103     | high     |

### Question

The query below is meant to report, for each site, the **total active contract revenue** and the **number of incidents**. It runs without errors — but the numbers are wrong.

**What is the problem? What would you do to fix it?**

```sql
SELECT
    s.site_id,
    s.name,
    SUM(c.base_fee)      AS total_revenue,
    COUNT(i.incident_id) AS incident_count
FROM sites s
JOIN contracts c ON c.site_id = s.site_id
JOIN incidents i ON i.site_id = s.site_id
WHERE c.status = 'active'
GROUP BY s.site_id, s.name;
```

---

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

## 05 · Latest reading per site `SQL`

### Schema

```
readings(
  reading_id  INT,
  site_id     INT,
  measured_at TIMESTAMP,
  signal_dbm  NUMERIC    -- signal strength in dBm
)
```

### Sample data

| reading_id | site_id | measured_at      | signal_dbm |
|------------|---------|------------------|------------|
| 1          | 101     | 2024-06-01 08:00 | -72        |
| 2          | 101     | 2024-06-01 14:00 | -68        |
| 3          | 101     | 2024-06-02 09:00 | -75        |
| 4          | 102     | 2024-06-01 10:00 | -80        |
| 5          | 102     | 2024-06-02 11:00 | -77        |
| 6          | 103     | 2024-06-01 07:00 | -65        |

### Question

Both queries below return the most recent signal reading for each site. **Which would you prefer in production, and why?**

#### Option A

```sql
SELECT reading_id, site_id, measured_at, signal_dbm
FROM readings r
WHERE measured_at = (
    SELECT MAX(measured_at)
    FROM readings
    WHERE site_id = r.site_id
);
```

#### Option B

```sql
SELECT reading_id, site_id, measured_at, signal_dbm
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY site_id ORDER BY measured_at DESC) AS rn
    FROM readings
) ranked
WHERE rn = 1;
```

---

## 06 · Two Sum `Python`

**Context:** given a list of integers and a target value, this function returns the indices of the two numbers that add up to the target. It is correct.

**Examples:**

```
Input:  nums = [2, 7, 11, 15], target = 9
Output: [0, 1]
Explanation: nums[0] + nums[1] == 9

Input:  nums = [3, 2, 4], target = 6
Output: [1, 2]

Input:  nums = [3, 3], target = 6
Output: [0, 1]
```

**What do you notice about it? How would you improve it?**

```python
def two_sum(nums, target):
    for i in range(len(nums)):
        for j in range(i + 1, len(nums)):
            if nums[i] + nums[j] == target:
                return [i, j]
    return []
```

---

## Database setup

### Exercise 01

```sql
CREATE TABLE contracts (
    contract_id INTEGER PRIMARY KEY,
    site_id     INTEGER NOT NULL,
    operator    TEXT NOT NULL,
    base_fee    NUMERIC NOT NULL,
    status      TEXT NOT NULL CHECK (status IN ('active', 'churned'))
);
```

```sql
INSERT INTO contracts (contract_id, site_id, operator, base_fee, status) VALUES
(1,  101, 'Claro', 8200, 'active'),
(2,  101, 'TIM',   4500, 'active'),
(3,  102, 'Claro', 6100, 'active'),
(4,  102, 'Vivo',  5300, 'active'),
(5,  103, 'TIM',   3800, 'active'),
(6,  103, 'TIM',   4200, 'active'),
(7,  103, 'TIM',   5900, 'active'),
(8,  104, 'Vivo',  9100, 'active'),
(9,  104, 'Claro', 2100, 'churned'),
(10, 105, 'Claro', 7400, 'active');
```

### Exercise 03

```sql
CREATE TABLE sites (
    site_id INTEGER PRIMARY KEY,
    name    TEXT NOT NULL
);

CREATE TABLE contracts (
    contract_id INTEGER PRIMARY KEY,
    site_id     INTEGER NOT NULL REFERENCES sites(site_id),
    operator    TEXT NOT NULL,
    base_fee    NUMERIC NOT NULL,
    status      TEXT NOT NULL CHECK (status IN ('active', 'churned'))
);

CREATE TABLE incidents (
    incident_id INTEGER PRIMARY KEY,
    site_id     INTEGER NOT NULL REFERENCES sites(site_id),
    severity    TEXT NOT NULL CHECK (severity IN ('critical', 'high', 'low'))
);
```

```sql
INSERT INTO sites VALUES (101, 'Alpha Tower'), (102, 'Beta Mast'), (103, 'Gamma Site');

INSERT INTO contracts VALUES
(1, 101, 'Claro', 8200, 'active'),
(2, 101, 'TIM',   4500, 'active'),
(3, 102, 'Vivo',  6100, 'active'),
(4, 103, 'Claro', 3800, 'active'),
(5, 103, 'TIM',   5200, 'active');

INSERT INTO incidents VALUES
(1, 101, 'critical'),
(2, 101, 'high'),
(3, 101, 'low'),
(4, 102, 'critical'),
(5, 103, 'high');
```

### Exercise 05

```sql
CREATE TABLE readings (
    reading_id  INTEGER PRIMARY KEY,
    site_id     INTEGER NOT NULL,
    measured_at TIMESTAMP NOT NULL,
    signal_dbm  NUMERIC NOT NULL
);
```

```sql
INSERT INTO readings VALUES
(1, 101, '2024-06-01 08:00', -72),
(2, 101, '2024-06-01 14:00', -68),
(3, 101, '2024-06-02 09:00', -75),
(4, 102, '2024-06-01 10:00', -80),
(5, 102, '2024-06-02 11:00', -77),
(6, 103, '2024-06-01 07:00', -65);
```
