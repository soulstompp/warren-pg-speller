# Changelog

## 0.1.0

The first release: a PostgreSQL planner hook that respells a SELECT through three rules before it
is planned, wherever the catalog proves them, whatever indexes the tables carry.

- A join a UNION ALL's arms share is taken out of the union and made once.
- A grouping over a join is taken on each side first, and the sides are met at the join key.
- The side the grouping does not count is grouped only over the keys the counted side holds.
