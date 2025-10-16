
---

# 6 — Autonomous Migration Tester (migration_tester.py)

**File:** `src/migration_tester.py`
```python
#!/usr/bin/env python3
"""
Autonomous Migration Tester - migration_tester.py

- Apply schema migration (simulated)
- Run sample queries (from a log-like file) and compare results/performance
- Produce a risk report (differences in outputs)
"""

import time
import random
import json
from typing import List, Callable, Any

# Simple in-memory DB as dict-of-tables
class InMemoryDB:
    def __init__(self, schema):
        self.schema = schema
        self.tables = {name: [] for name in schema.keys()}

    def insert(self, table, row):
        self.tables[table].append(row)

    def query(self, func: Callable[[dict], bool]):
        # run filter function on every row across tables (demo)
        out = []
        for t, rows in self.tables.items():
            for r in rows:
                try:
                    if func(r):
                        out.append((t, r))
                except:
                    pass
        return out

def run_migration_test(old_schema, new_schema, seed_queries: List[Callable[[dict], bool]]):
    old_db = InMemoryDB(old_schema)
    new_db = InMemoryDB(new_schema)
    # populate with synthetic rows
    for _ in range(500):
        old_db.insert("users", {"id": random.randint(1,1000), "name": f"user{random.randint(1,100)}", "active": random.choice([True, False])})
        new_db.insert("users", {"id": random.randint(1,1000), "username": f"user{random.randint(1,100)}", "active": random.choice([True, False])})
    report = []
    for q in seed_queries:
        t0 = time.time(); res_old = old_db.query(q); t1 = time.time()
        t2 = time.time(); res_new = new_db.query(q); t3 = time.time()
        diff = len(res_old) - len(res_new)
        report.append({
            "query": q.__name__,
            "old_count": len(res_old),
            "new_count": len(res_new),
            "old_time_ms": (t1-t0)*1000,
            "new_time_ms": (t3-t2)*1000,
            "count_diff": diff
        })
    return report

# demo queries
def has_active(r): return r.get("active") is True
def name_contains_user1(r): return "user1" in (r.get("name") or r.get("username") or "")

def main():
    old_schema = {"users": ["id", "name", "active"]}
    new_schema = {"users": ["id", "username", "active", "created_at"]}
    report = run_migration_test(old_schema, new_schema, [has_active, name_contains_user1])
    print(json.dumps(report, indent=2))

if __name__ == "__main__":
    main()
