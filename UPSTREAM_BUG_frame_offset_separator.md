# Bug: bounded window frame offsets emit malformed SQL (`31PRECEDING`)

**Affects:** 1.0.2 (also present in 0.32.7)
**Severity:** generated SQL is rejected by the server
**Found:** 2026-08-24, building a windowed aggregate against PostgreSQL

## Summary

`Frame::Preceding(n)` and `Frame::Following(n)` write the offset and the
keyword with **no separator between them**, producing `31PRECEDING` (inline
builder) or `$7PRECEDING` (parameterised builder). Both are syntax errors.

## Reproduction

```rust
let mut w = WindowStatement::new();
w.order_by(Alias::new("bucket_start"), Order::Asc);
w.frame(FrameType::Rows, Frame::Preceding(31), Some(Frame::Preceding(1)));

let (sql, values) = Query::select()
    .expr_as(Expr::cust("avg(volume) OVER w"), Alias::new("adv"))
    .from(Alias::new("s"))
    .window(Alias::new("w"), w)
    .to_owned()
    .build(PostgresQueryBuilder);
```

Generated:

```sql
WINDOW "w" AS ( ORDER BY "bucket_start" ASC ROWS BETWEEN $7PRECEDING AND $8PRECEDING )
```

PostgreSQL:

```
ERROR:  trailing junk after parameter at or near "$7PRECEDING"
```

Expected:

```sql
WINDOW "w" AS ( ORDER BY "bucket_start" ASC ROWS BETWEEN $7 PRECEDING AND $8 PRECEDING )
```

## Cause

`src/backend/query_builder.rs`, `prepare_frame`:

```rust
Frame::Preceding(v) => {
    self.prepare_value(v.into(), sql);
    sql.write_str("PRECEDING").unwrap();   // no leading space
}
...
Frame::Following(v) => {
    self.prepare_value(v.into(), sql);
    sql.write_str("FOLLOWING").unwrap();   // same
}
```

The sibling arms are unaffected because each writes a complete, correctly
spaced string of its own:

```rust
Frame::UnboundedPreceding => sql.write_str("UNBOUNDED PRECEDING").unwrap(),
Frame::CurrentRow         => sql.write_str("CURRENT ROW").unwrap(),
Frame::UnboundedFollowing => sql.write_str("UNBOUNDED FOLLOWING").unwrap(),
```

## Fix

```diff
 Frame::Preceding(v) => {
     self.prepare_value(v.into(), sql);
-    sql.write_str("PRECEDING").unwrap();
+    sql.write_str(" PRECEDING").unwrap();
 }
 Frame::Following(v) => {
     self.prepare_value(v.into(), sql);
-    sql.write_str("FOLLOWING").unwrap();
+    sql.write_str(" FOLLOWING").unwrap();
 }
```

## Why this survived

`Frame::Preceding` / `Frame::Following` appear **nowhere** in `tests/` -
`grep -rn PRECEDING tests/` returns zero hits - and every doc example on
`WindowStatement::frame_start` / `frame_between` uses only
`UnboundedPreceding`, `UnboundedFollowing` or `CurrentRow`, which are the three
variants that cannot reach this path.

So the regression test matters as much as the one-character fix. Suggested
coverage, in the style of the existing builder tests:

- `ROWS BETWEEN 31 PRECEDING AND 1 PRECEDING` (bounded both ends)
- `RANGE BETWEEN 1 PRECEDING AND 1 FOLLOWING` (both keywords, both directions)
- `ROWS 5 PRECEDING` (frame_start only)
- the same three through a PARAMETERISED build, asserting `$1 PRECEDING` and
  not `$1PRECEDING`, since the inline and bound paths format differently

## Note for the microscaler checkout

`~/Workspace/microscaler/sea-query` sits on `0.32.7` while the workspaces build
against crates.io `1.0.2`. The bug is in both, but the checkout is NOT what
PriceWhisperer and hauliage compile against - bring it to 1.0.2 before
patching, or the fix will not be the one being consumed.
