# SQL

---

## 01 · Active contract filtering

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

---

#### Option A

```sql
SELECT operator, AVG(base_fee) AS avg_fee
FROM contracts
WHERE status = 'active' AND AVG(base_fee) > 5000
GROUP BY operator
HAVING COUNT(*) >= 3
ORDER BY avg_fee DESC;
```

---

#### Option B

```sql
SELECT operator, AVG(base_fee) AS avg_fee
FROM contracts
WHERE status = 'active'
GROUP BY operator
HAVING COUNT(*) >= 3 AND AVG(base_fee) > 5000
ORDER BY avg_fee DESC;
```

---

#### Option C

```sql
SELECT operator, AVG(base_fee) AS avg_fee
FROM contracts
WHERE status = 'active' AND base_fee > 5000
GROUP BY operator
HAVING COUNT(*) >= 3
ORDER BY avg_fee DESC;
```
