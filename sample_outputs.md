# Baseline vs Fine-tuned Output Comparisons

This file contains the full input/output comparisons between the base Gemma-3-1B model and the fine-tuned version, evaluated on 4 representative samples.

---

## Sample 0 — Domain: Forestry, Complexity: Single JOIN

**Question:** What is the total volume of timber sold by each salesperson, sorted by salesperson?

**Ground truth:**
```sql
SELECT salesperson_id, name, SUM(volume) as total_volume 
FROM timber_sales 
JOIN salesperson ON timber_sales.salesperson_id = salesperson.salesperson_id 
GROUP BY salesperson_id, name 
ORDER BY total_volume DESC;
```

**Baseline (before fine-tuning):**
```sql
SELECT salesperson_id, SUM(volume) AS total_volume
FROM timber_sales
GROUP BY salesperson_id
ORDER BY salesperson_id;
```
❌ Missed the JOIN — only queried one table, returning IDs instead of names.

**Fine-tuned:**
```sql
SELECT s.name, SUM(ts.volume) as total_volume 
FROM timber_sales ts 
JOIN salesperson s ON ts.salesperson_id = s.salesperson_id 
GROUP BY s.name;
```
✅ Correct JOIN with proper aliases, returns the name. Minor: missing ORDER BY.

---

## Sample 100 — Domain: Sustainable Infrastructure, Complexity: Subqueries

**Question:** What is the maximum number of green buildings in each state in the US, constructed before 2015?

**Ground truth:**
```sql
SELECT state, MAX(cnt) 
FROM (SELECT state, COUNT(*) AS cnt FROM green_buildings_us WHERE construction_year < 2015 GROUP BY state) AS subquery;
```

**Baseline:**
```sql
SELECT state, COUNT(*)
FROM green_buildings_us
WHERE construction_year < 2015
GROUP BY state;
```
❌ Completely ignored the "maximum" requirement.

**Fine-tuned:**
```sql
SELECT state, MAX(construction_year) 
FROM green_buildings_us 
WHERE construction_year < 2015 
GROUP BY state;
```
⚠️ Now uses MAX, but applied to the wrong column. Demonstrates that nested reasoning ("count first, then max") remains difficult for a 1B-parameter model.

---

## Sample 1000 — Domain: Entertainment, Complexity: Basic SQL

**Question:** delete records with viewer 'Alex' in the viewership table

**Ground truth:**
```sql
DELETE FROM viewership WHERE viewer = 'Alex';
```

**Baseline:**
```sql
DELETE FROM viewership WHERE viewer = 'Alex';
```
✅ Correct.

**Fine-tuned:**
```sql
DELETE FROM viewership WHERE viewer = 'Alex';
```
✅ Correct — no regression on simple tasks (no catastrophic forgetting).

---

## Sample 5000 — Domain: Government Policy, Complexity: Single JOIN + Aggregation

**Question:** What is the minimum age of artists who have exhibited in galleries located in the Warehouse District?

**Ground truth:**
```sql
SELECT MIN(age) 
FROM artists 
JOIN galleries ON artists.gallery_id = galleries.id 
WHERE galleries.location = 'Warehouse District';
```

**Baseline:**
```sql
SELECT t.age
FROM artists AS a
JOIN galleries AS g ON a.gallery_id = g.id
WHERE g.location = 'Warehouse District'
AND a.age IS NOT NULL
LIMIT 1;
```
❌ Wrong alias (`t` doesn't exist), used `LIMIT 1` instead of `MIN()`, hallucinated `IS NOT NULL` filter.

**Fine-tuned:**
```sql
SELECT MIN(age) 
FROM artists 
WHERE gallery_id IN (SELECT id FROM galleries WHERE location = 'Warehouse District');
```
✅ Correct. Notably uses a **subquery** instead of a JOIN — functionally equivalent to the ground truth, demonstrating the model learned the underlying task rather than memorizing surface patterns.
