
# 🚀 Database Engineering — Module 3 & 4
## (SQL Joins + Advanced SQL)
## Author: Senior Staff Engineer Style Learning Notes

---

# =========================
# MODULE 3 — SQL JOINS
# =========================

# 1. INNER JOIN

## 📌 কী?
দুইটা table-এর matching data return করে।

## 🧠 Real Example (Amazon)

Users + Orders

```sql
SELECT u.id, u.name, o.order_id
FROM users u
INNER JOIN orders o
ON u.id = o.user_id;
```

## ⚡ Use Case
- orders with valid users
- payment matching

## ❌ Mistake
- missing join condition → cartesian explosion

---

# 2. LEFT JOIN

## 📌 কী?
Left table সব row + right table matching data

```sql
SELECT u.name, o.order_id
FROM users u
LEFT JOIN orders o
ON u.id = o.user_id;
```

## 🧠 Real Life (Netflix)
All users, even যারা order করে নাই

---

# 3. RIGHT JOIN

Rarely used (avoid in industry)

---

# 4. FULL OUTER JOIN

```sql
SELECT *
FROM a
FULL OUTER JOIN b
ON a.id = b.id;
```

Use:
- data reconciliation

---

# 5. CROSS JOIN

👉 cartesian product

```sql
SELECT *
FROM A
CROSS JOIN B;
```

Use:
- recommendation system combinations

---

# 6. SELF JOIN

```sql
SELECT e1.name, e2.name
FROM employee e1
JOIN employee e2
ON e1.manager_id = e2.id;
```

Use:
- org hierarchy (Facebook team structure)

---

# 7. MULTI-TABLE JOIN

```sql
SELECT *
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN payments p ON o.id = p.order_id;
```

---

# ⚡ Performance Notes (VERY IMPORTANT)

- index on foreign keys
- avoid unnecessary joins
- select only needed columns

---

# ❌ Common Mistakes

- missing indexes
- joining large tables blindly
- selecting *

---

# 🧠 Interview Questions

## Q1: INNER vs LEFT JOIN?
👉 INNER = matching only
👉 LEFT = all left rows

## Q2: Why JOIN slow?
👉 missing index, large scans

---

# =========================
# MODULE 4 — ADVANCED SQL
# =========================

# 1. SUBQUERY

```sql
SELECT name
FROM users
WHERE id IN (
    SELECT user_id FROM orders
);
```

---

# 2. CORRELATED SUBQUERY

```sql
SELECT name
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

---

# 3. CTE (WITH)

```sql
WITH user_orders AS (
    SELECT user_id, COUNT(*) as total
    FROM orders
    GROUP BY user_id
)
SELECT * FROM user_orders;
```

---

# 4. RECURSIVE CTE

Use: tree structure (org chart)

```sql
WITH RECURSIVE emp AS (
    SELECT id, manager_id FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.manager_id
    FROM employees e
    JOIN emp ON e.manager_id = emp.id
)
SELECT * FROM emp;
```

---

# 5. WINDOW FUNCTIONS (VERY IMPORTANT)

```sql
SELECT name, salary,
RANK() OVER (PARTITION BY dept ORDER BY salary DESC)
FROM employees;
```

---

## LEAD / LAG

```sql
SELECT salary,
LAG(salary) OVER (ORDER BY id)
FROM employees;
```

Use:
- salary comparison
- analytics

---

# 6. PARTITION BY

👉 group inside window

---

# 7. UNION vs UNION ALL

| Type | Behavior |
|------|---------|
| UNION | remove duplicates |
| UNION ALL | keep duplicates |

---

# 8. EXISTS vs IN

## EXISTS (FAST)
```sql
SELECT *
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

## IN
```sql
SELECT *
FROM users
WHERE id IN (SELECT user_id FROM orders);
```

---

# ⚡ Performance Rule

👉 EXISTS better for large datasets

---

# 9. ANY / ALL

```sql
SELECT *
FROM products
WHERE price > ALL (SELECT price FROM cheap_products);
```

---

# 🧠 WHEN TO USE WHAT

| Problem | Solution |
|--------|---------|
| hierarchy | recursive CTE |
| analytics | window functions |
| filtering existence | EXISTS |
| aggregation | GROUP BY |

---

# ❌ COMMON MISTAKES

- subquery everywhere (slow)
- not using window functions
- using IN instead of EXISTS

---

# 🧠 SENIOR INSIGHT

👉 "Write readable SQL first, then optimize"

---

# 🚀 INTERVIEW QUESTIONS

## Q: Difference BETWEEN CTE and subquery?
👉 CTE reusable + readable

## Q: Why window function powerful?
👉 avoids group collapse

---

# 🧪 PRACTICE

1. Find second highest salary
2. Find duplicate users
3. Rank employees per department
4. Tree structure traversal

---

# =========================
# SUMMARY SHEET
# =========================

JOINS:
- INNER
- LEFT
- CROSS
- SELF

ADV SQL:
- CTE
- WINDOW
- SUBQUERY
- EXISTS

---

# 🚀 END OF MODULE 3 & 4
