# PR #2 — feat: add list stats endpoint

**Author:** Maya Okafor
**Branch:** `feat/list-stats`
**Touches:** `routes/lists.py`, `services/list_service.py`

---

## Background

The frontend team asked for a stats endpoint to power the active shopping view.
Their exact request:

> "We need to show users how many items are still on their list and break down
> what's remaining by category — so someone shopping at a grocery store can see
> 'I still need 2 things in produce, 1 in dairy' and navigate by section."

## What this does

Adds `GET /lists/<list_id>/stats` — returns total items, purchased count,
remaining count, and a per-category breakdown.

Prompt I used to generate the initial implementation:

> "Write a Flask route that returns grocery list statistics including total item
> count, purchased count, remaining count, and a breakdown of item counts by
> category. Use SQLAlchemy to query the database."

Tested on both seeded lists and the numbers add up.

## Files changed

See `pr2_list_stats.py` for the full proposed code.

## Testing done

```bash
python seed_data.py
python try_prs.py

curl http://127.0.0.1:5000/lists/LIST_ID/stats
# → {
#     "list_id": "...",
#     "total_items": 8,
#     "purchased": 3,
#     "remaining": 5,
#     "by_category": {
#       "produce": 2, "dairy": 2, "bakery": 1, "meat": 1, "pantry": 2
#     }
#   }
```
# PR #2 — feat: add list stats endpoint

## Issues Found

---

### 1. Missing list existence validation

**File:** `services/list_service.py`
**Function:** `get_list_stats(list_id)`

**What’s wrong:**
The function directly queries:

```python id="p9k3x1"
items = Item.query.filter_by(list_id=list_id).all()
```

without verifying the list actually exists.

**Why this matters in production:**

* Invalid `list_id` returns a successful response with empty stats
* Frontend cannot distinguish between “empty list” vs “invalid list”
* Leads to misleading UI states (e.g., showing a valid empty shopping session)

**Concrete fix:**

```python id="l2q8sa"
grocery_list = GroceryList.query.get(list_id)
if not grocery_list:
    raise ValueError("List not found")
```

Then handle in route as `404`.

---

### 2. Silent data ambiguity for missing categories

**File:** `services/list_service.py`
**Function:** `get_list_stats(list_id)`

**What’s wrong:**
Category grouping uses:

```python id="c7xq2p"
cat = item.category or "uncategorized"
```

This silently collapses all missing or malformed categories into a single bucket.

**Why this matters in production:**

* Makes data quality issues invisible
* Masks backend validation problems
* Can distort analytics (e.g., too many “uncategorized” items hides real categorization bugs)

**Concrete fix:**
At minimum, distinguish missing vs invalid:

```python id="v8m1dd"
if item.category is None:
    cat = "uncategorized"
elif item.category.strip() == "":
    cat = "invalid_category"
else:
    cat = item.category
```

Even better: enforce category validation at item creation time.

---

### 3. Inefficient full table scan (N+1 risk in larger datasets)

**File:** `services/list_service.py`

**What’s wrong:**
This line loads all items into memory:

```python id="f4r9xk"
items = Item.query.filter_by(list_id=list_id).all()
```

Then computes aggregates in Python.

**Why this matters in production:**

* Poor scalability for large lists (hundreds/thousands of items)
* Unnecessary memory usage
* Moves aggregation work from DB (optimized) to Python (slower)

**Concrete fix:**
Push aggregation into SQL:

```python id="q2v8lm"
from sqlalchemy import func

rows = db.session.query(
    Item.category,
    func.count(Item.id),
    func.sum(Item.is_purchased.cast(db.Integer))
).filter_by(list_id=list_id).group_by(Item.category).all()
```

---

### 4. Potential mismatch between `remaining` and actual item state

**File:** `services/list_service.py`

**What’s wrong:**
Remaining is computed as:

```python id="k9p2aa"
remaining = total - purchased
```

This assumes:

* every item is either purchased or not
* no NULL or inconsistent states exist

**Why this matters in production:**
If data ever gets corrupted (e.g., `is_purchased = NULL`):

* counts become inaccurate
* remaining may not match actual items when queried separately
* UI may show inconsistent numbers vs item list endpoint

**Concrete fix:**
Compute remaining explicitly:

```python id="t6q1zz"
remaining = sum(1 for item in items if not item.is_purchased)
```

(or enforce strict boolean constraints at DB level)

---

### 5. No handling of missing/invalid list access in route

**File:** `routes/lists.py`
**Function:** `list_stats(list_id)`

**What’s wrong:**
Route assumes service always succeeds:

```python id="w3n9kk"
stats = list_service.get_list_stats(list_id)
return jsonify(stats), 200
```

No error handling for invalid list IDs.

**Why this matters in production:**

* API always returns 200 even for invalid IDs
* Clients cannot distinguish success vs error
* Breaks REST semantics

**Concrete fix:**

```python id="r8m3pp"
try:
    stats = list_service.get_list_stats(list_id)
    return jsonify(stats), 200
except ValueError as e:
    return jsonify({"error": str(e)}), 404
```

---

### 6. Category grouping is case-sensitive (data fragmentation risk)

**File:** `services/list_service.py`

**What’s wrong:**
Categories are grouped exactly as stored:

```python id="z2x8cc"
by_category[cat] = by_category.get(cat, 0) + 1
```

So "Dairy", "dairy", and "DAIRY" become separate buckets.

**Why this matters in production:**

* UI shows fragmented categories
* Users see inconsistent store grouping
* Analytics becomes unreliable

**Concrete fix:**
Normalize category names:

```python id="n1v7qq"
cat = (item.category or "uncategorized").strip().lower()
```

---

## Questions for the author

1. Should `/stats` include only unpurchased items breakdown or all items?
2. Do we want category normalization enforced at write-time instead of read-time?
3. Should this endpoint support caching for performance (stats don’t change per request often)?
4. Should invalid list IDs return 404 instead of empty stats?
5. Do we expect categories to be user-defined or from a fixed enum?

---
