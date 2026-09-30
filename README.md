# warren-pg-speller

Some queries are slow only because of how they are written. Join purchases to the lines of the sets
they bought and count afterwards, and PostgreSQL builds every pair of a purchase and a line before it
can count anything. Count the purchases of each set, and the lines of each set, and join the two
counts, and the answer is the same while the join pairs sets instead of rows.

warren-pg-speller makes that second spelling for you, inside the planner. A statement is planned
through another spelling of it that returns the same rows, and only where the statement and the
catalog prove that the two agree. Where anything is in doubt, the statement is planned exactly as it
was written.

It is a Rust library for [pgrx](https://github.com/pgcentralfoundation/pgrx) extensions: an extension
calls one function from `_PG_init`, and PostgreSQL's `planner_hook` does the rest. Each of its three
rules is written out below as what it rewrites and when, so that a database engine can take any of
them into its own planner.

## Using it

```toml
[dependencies]
pgrx = "=0.19.2"
warren-pg-speller = { path = "../warren-pg-speller", default-features = false }

[features]
pg18 = ["pgrx/pg18", "warren-pg-speller/pg18"]
```

```rust
#[pg_guard]
pub extern "C-unwind" fn _PG_init() {
    warren_pg_speller::init();
}
```

`init` puts the hook in place after any planner hook already installed. A statement is respelled only
when the extension's library was loaded before the statement was planned, so load it the way any
planner hook is loaded: `shared_preload_libraries` for every session, `session_preload_libraries` for
some, or `LOAD` in one.

To see it work, read the plan. A respelled grouping shows a WITH query named `fold_a`, one side's
grouping, read twice: where the two sides meet, and by the other side's grouping, which keeps only
the keys `fold_a` holds.

```sql
EXPLAIN (COSTS OFF)
SELECT b.zone, l.part, count(DISTINCT p.id), sum(l.qty)
FROM purchases p
JOIN holdings h ON h.builder_id = p.builder_id AND h.row_no = p.row_no
JOIN builders b ON b.builder_id = h.builder_id
JOIN lines l    ON l.set_num = h.set_num
GROUP BY 1, 2;
```

## What it keeps, whatever it rewrites

- A spelling is built from the statement as PostgreSQL parsed it, never from its text.
- It reads no table the statement does not read, and reads each one as the statement does: as the
  owner of the view it lies in, or as the user running the statement, under the same row security.
- The indexes a table carries play no part in any rule. A plan made from a spelling depends on the
  unique keys it rests on, so it is made again when an index on a table it reads changes.
- Only inner joins are read. A query that reads a column of a query around it, reads a relation
  beside it laterally, or calls a volatile function is left as written.
- Every query level is read: subqueries in FROM, WITH queries that are not recursive, and subqueries
  in expressions, each before the query around it.
- ORDER BY, LIMIT and OFFSET stay on the outside of every spelling.

## Rule 1: a join distributes over UNION ALL

**Identity.** (A ⋈ B₁) ∪ (A ⋈ B₂) = A ⋈ (B₁ ∪ B₂), for UNION ALL.

**What it rewrites.** A UNION ALL whose arms each join the same relation A, in the same way, to
something else becomes A joined once to a UNION ALL of the rest of each arm.

```sql
-- as written: the purchases are read by each arm
SELECT bt.id, i.id AS inventory_id FROM bought bt JOIN inventories i ON i.set_num = bt.set_num
UNION ALL
SELECT bt.id, n.inventory_id       FROM bought bt JOIN nested n      ON n.set_num = bt.set_num

-- respelled: the purchases are read once
SELECT bt.id, arms.inventory_id
FROM bought bt
JOIN (SELECT i.set_num AS key1, i.id AS inventory_id FROM inventories i
      UNION ALL
      SELECT n.set_num, n.inventory_id FROM nested n) arms ON arms.key1 = bt.set_num
```

**When.** Every condition below holds, checked on the parsed statement and the catalog.

1. The statement's set operations are UNION ALL and nothing else, with at least two arms, and the
   statement locks no rows.
2. Every arm is a plain SELECT over inner joins: no grouping, aggregate, window function, DISTINCT,
   ORDER BY, LIMIT or OFFSET, set operation, row lock, WITH query of its own, set-returning function
   in its target list, or volatile function. It reads no column of a query around it, and returns no
   hidden column.
3. Every FROM item of every arm is a table, a WITH query that is not the recursive reference, or a
   subquery, and none is LATERAL.
4. Exactly one FROM item of each arm reads the same rows as a FROM item of the first arm:
   - the same table, with the same inheritance setting and no TABLESAMPLE; or the same WITH query at
     the same level; or equal subqueries with the same security barrier and view;
   - read as the same role, needing the same permissions;
   - under equal row security conditions.
5. Each arm keeps at least one other FROM item.
6. Each arm's conditions on A alone are the same in every arm.
7. Each condition reading A and the rest of the arm is an equality between an expression of A alone
   and an expression of the rest alone, by the default equality of both sides' type, under a
   collation that is deterministic or none. Every arm has the same such equalities: the same
   expression of A, on the same side of the operator, with the same operator and collation, and the
   rest's side of the same type, type modifier and collation.
8. Each column the arms return reads A alone, the same expression in every arm and of the type the
   union returns, or does not read A.

Where several relations qualify, the first of the first arm's FROM items that does is taken.

**The spelling.** The new UNION ALL's arms return the rest's side of each equality (`key1`, `key2`, …)
and then each column that does not read A. The outer query reads A and that union, under A's own
conditions and the equalities with A's side put back, and returns A's columns and the union's in the
original order. A WITH query that every arm read as A is now read once, so PostgreSQL may read it in
place.

## Rule 2: group before the join, at the cut

**Identity.** Grouping A ⋈ₖ B by G is grouping, by G, the join at k of A grouped by (G_A, k) and B
grouped by (G_B, k), where G_A and G_B are the group keys each side holds, each aggregate being taken
on its own side first and combined over the join.

**What it rewrites.** A grouped query whose relations split into two sides that meet only at one set
of equal columns, the key. Each side is grouped by its own group keys and the key, and the two
groupings are joined at the key.

```sql
-- as written: every purchase meets every line of its set before anything is counted
SELECT b.zone, l.part, count(DISTINCT p.id), sum(l.qty)
FROM purchases p
JOIN holdings h ON h.builder_id = p.builder_id AND h.row_no = p.row_no
JOIN builders b ON b.builder_id = h.builder_id
JOIN lines l    ON l.set_num = h.set_num
GROUP BY 1, 2

-- respelled: the purchases counted by set and zone, the lines summed by set and part
WITH fold_a AS (
  SELECT b.zone AS group1, h.set_num AS key, count(*) AS rows
  FROM purchases p
  JOIN holdings h ON h.builder_id = p.builder_id AND h.row_no = p.row_no
  JOIN builders b ON b.builder_id = h.builder_id
  GROUP BY 1, 2)
SELECT fold_a.group1, fold_b.group2,
       sum(fold_a.rows)::bigint, sum(fold_b.fold2 * fold_a.rows)::bigint
FROM fold_a
JOIN (SELECT l.part AS group2, l.set_num AS key, sum(l.qty) AS fold2
      FROM lines l
      WHERE l.set_num IN (SELECT key FROM fold_a)
      GROUP BY 1, 2) fold_b ON fold_a.key = fold_b.key
GROUP BY 1, 2
```

**When.** Every condition below holds, checked on the parsed statement and the catalog.

1. The query is a SELECT with GROUP BY and aggregates, and without grouping sets, window functions,
   set-returning functions in its target list, set operations, row locks, a data-modifying WITH
   query, a subquery in its target list or HAVING, or a volatile function, and it reads no column or
   aggregate of a query around it.
2. Its relations are read side by side:
   - every join is an inner join, and the join conditions and WHERE together are its conditions;
   - a WITH query that only this query reads, read once, that is a SELECT, not recursive, not
     `MATERIALIZED` and free of volatile functions, is read in its reader's place;
   - a subquery in FROM that is not LATERAL, not a view, not behind a security barrier or row
     security, and is itself a plain SELECT over inner joins (rule 1, condition 2), is read as the
     relations it joins;
   - after that there are at least two FROM items, each a table, a WITH query or a subquery, none
     LATERAL.
3. The key is a set of columns the query's equalities make equal (each equality by the default
   equality of both sides' type, under a deterministic collation or none), of at least two FROM items.
4. The cut: the FROM items are joined into parts by every condition other than an equality of the
   key, and by what each group key and each aggregate reads. There are at least two parts. Side A is
   the part, or parts, holding each `count(DISTINCT u)`'s table, or else the first aggregate of a
   column; side B is the rest.
   - Every group key reads a column, and reads one side.
   - Every aggregate reads one side (`count(*)` reads none).
   - Each side carries an aggregate of a column of its own.
   - The key has a column on each side.
5. The aggregates are `count(*)`, `count(x)`, `sum(x)` of `int2`, `int4`, `int8` or `numeric`,
   `min(x)`, `max(x)` and `count(DISTINCT u)`, each from `pg_catalog`, with no FILTER, no ORDER BY
   inside it, not variadic and not an ordered-set aggregate, and each but `count(*)` reads a column. A
   `sum` of a floating-point type is left as written.
6. For each `count(DISTINCT u)`:
   - `u` is a single column, NOT NULL, and alone a unique key of its table;
   - from `u`'s table the key is reached through equalities, each of which covers a unique key of
     the table it joins, every column of that key NOT NULL.

   A unique key is a valid, immediate, whole-table unique B-tree index on plain columns, each under its
   type's default operator family. A table read with inheritance children (not a partitioned table),
   or with TABLESAMPLE, has none.
7. Among several cuts that pass, the one whose side A has the fewest FROM items is taken.

**The spelling.** Each side becomes a query over its own FROM items and conditions, grouped by its
group keys and the key, returning a count of its rows where the other side's aggregates need it. The
two are joined at the key, and the query's own GROUP BY, HAVING, DISTINCT, ORDER BY and LIMIT are
taken over them, each aggregate replaced:

| as written, on side S | over the grouped sides |
|---|---|
| `count(*)` | `sum(rows_A × rows_B)` |
| `count(x)`, `sum(x)` | `sum(S's value × the other side's rows)` |
| `min(x)`, `max(x)` | `min` or `max` of S's value |
| `count(DISTINCT u)`, every table of side A reached by a unique key | `sum(rows_A)` |
| `count(DISTINCT u)`, otherwise | `sum` of side A's `count(DISTINCT u)` |

Each sum is taken as `numeric` and cast back to the aggregate's own type, so it is exact.

## Rule 3: each side keeps only the keys the other holds

**Identity.** B ⋈ₖ A = (B ⋉ₖ πₖ A) ⋈ₖ A, for an inner join at k.

**What it rewrites.** In rule 2's spelling, side A's grouping becomes a WITH query, named `fold_a` (or
`fold_a_2`, … where the statement already has a WITH query of that name), read by the join of the two
sides. Side B's grouping gains the condition `k IN (SELECT key FROM fold_a)`, so B is grouped only
over the keys A holds.

**When.** Wherever rule 2 applies. The join of the two sides is an inner join, so a row of B whose key
A does not hold meets nothing, and dropping it before grouping changes no row of the answer.

## Where its laws run

The rules' laws are `#[pg_test]`s, and pgrx runs a crate's `#[pg_test]`s only in the extension it
builds. They run in `pg_warrendex`, the extension that installs the hook, in its
`src/respelling/` directory: one module for each rule, one for a question that reaches all three,
and one for what a respelled statement reads and as whom.

## Requirements

PostgreSQL 18 and pgrx 0.19.2. The `pg18` feature is the default; `pg19` builds the hook for the next
planner signature.

## License

Copyright (c) 2026 Kenneth Allen Flegal

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))
- MIT license ([LICENSE-MIT](LICENSE-MIT))

at your option.

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the
work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any
additional terms or conditions.
