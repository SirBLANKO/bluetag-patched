# BlueTag: SQL injection in board search

**Location:** The searchItems() function in the items route.

**Problem:** Search text (q), category, and kind are inserted directly into SQL strings. Specially crafted input can change the query's conditions.

**Impact:** A visitor can manipulate search results and bypass filters. The vulnerable query can also bypass the condition that excludes listings marked removed.

## Steps to reproduce:

1. Open the locally running board.
2. Set the category and kind filters to "all."
3. Search for something not on the board and confirm it produces no matches.
4. Search for that same thing but add `OR 1=1 --` to the end of it.
5. Now you can see everything, even items that were already resolved.

**Expected behavior:** The app searches for the supplied text and returns no matches.

**Observed behavior before the patch:** ![Before patch screenshot](./Screenshot%202026-09-13%20135343.png)

Before the patch, entering a SQL injection payload like `OR 1=1 --` into the search bar caused the page to return all matches, including items that were already resolved and taken down from the board.

**Cause:** User input becomes part of the SQL command before the command is prepared, allowing attackers to inject arbitrary SQL logic.

## Code Examples

### Vulnerable Code (Before Patch)
```javascript
function searchItems({ q, category, kind }) {
  let sql = `
    SELECT
      items.id,
      items.user_id,
      items.kind,
      items.category,
      items.title,
      items.description,
      items.location,
      items.contact_pref,
      items.status,
      items.created_at,
      users.display_name AS owner_name
    FROM items
    JOIN users ON users.id = items.user_id
    WHERE items.status != 'removed'
  `;

  if (q) {
    sql += ` AND items.title || ' ' || items.description || ' ' || items.location LIKE '%${q}%'`;
  }

  if (category && category !== "all") {
    sql += ` AND items.category = '${category}'`;
  }

  if (kind && kind !== "all") {
    sql += ` AND items.kind = '${kind}'`;
  }

  sql += " ORDER BY items.created_at DESC LIMIT 50";
  return db.prepare(sql).all();  // No parameters passed - SQL injection vulnerability!
}
```

In this vulnerable version, user input is directly interpolated into the SQL string using template literals. An attacker can enter `' OR '1'='1` as the search term (q), which changes the query to:
```sql
WHERE items.status != 'removed' AND items.title || ' ' || items.description || ' ' || items.location LIKE '%' OR '1'='1%'
```

**Why this works:** The `'1'='1'` comparison is always true in SQL. By injecting this into the query, the attacker effectively replaces the LIKE condition with an always-true expression. This causes the database to return all rows instead of filtering based on the search term. Combined with the `--` comment operator, the attacker can also remove any remaining filters. In this case, the result bypasses all filters and exposes resolved items.

### Patched Code (After Fix)
```javascript
function searchItems({ q, category, kind }) {
  let sql = `
    SELECT
      items.id,
      items.user_id,
      items.kind,
      items.category,
      items.title,
      items.description,
      items.location,
      items.contact_pref,
      items.status,
      items.created_at,
      users.display_name AS owner_name
    FROM items
    JOIN users ON users.id = items.user_id
    WHERE items.status != 'removed'
  `;

  const params = [];

  if (q) {
    sql += `
      AND (
        items.title || ' ' ||
        items.description || ' ' ||
        items.location
      ) LIKE ?
    `;
    params.push(`%${q}%`);
  }

  if (category && category !== "all") {
    sql += " AND items.category = ?";
    params.push(category);
  }

  if (kind && kind !== "all") {
    sql += " AND items.kind = ?";
    params.push(kind);
  }

  sql += " ORDER BY items.created_at DESC LIMIT 50";
  return db.prepare(sql).all(...params);  // Parameters passed separately - safe!
}
```

The patched version uses `?` placeholders and passes values separately via `.all(...params)`. This ensures user input is treated as data, not SQL code. Even if an attacker enters `' OR '1'='1`, it will be safely escaped as a literal string and won't break the SQL logic.

**Fix:** In the searchItems() function, replace all interpolated values with `?` placeholders for the search term (q), category, and kind parameters. Pass their values separately through the `.all(...params)` method to use prepared statements.

**Observed behavior after the patch:** ![After patch screenshot](./Screenshot%202026-09-13%20134213.png)

After the patch, entering SQL injection payloads into the search bar no longer affects the query results. The SQL injection was successfully prevented, and the search function now behaves normally, returning only legitimate matches for the entered search term.

## Normal functionality checks:

| Test | Expected behavior | Actual result |
|---|---|---|
| Injected search | No matches or SQL error after patch | No matches - SQL injection prevented |
| Search for a known item | Relevant listing appears | Relevant listing appears |
| Category and kind filters together | Both filters apply | Both filters apply |
| Search containing an apostrophe | Search runs without a SQL error | Search runs without a SQL error |
| Register, sign in, and sign out | Each action works | Each action works |
| Create and view a post | Post is saved and displayed | Post is saved and displayed |
| Resolve your own post | Post becomes resolved | Post becomes resolved |

## Summary

This SQL injection vulnerability in the BlueTag board search function has been successfully patched. The fix converts the dynamic SQL query construction to use prepared statements with parameterized queries, which is the industry standard for preventing SQL injection attacks.

**Key improvements:**
- User input is no longer directly interpolated into SQL strings using template literals
- All three vulnerable parameters (q, category, kind) now use `?` placeholders
- Values are passed as separate parameters to the database driver, ensuring they're treated as data only
- The patch maintains full functionality while eliminating the security risk

**Testing results confirm:**
- SQL injection payloads are no longer effective
- All normal search and filtering functionality works as expected
- User account operations (registration, login, logout) remain secure
- Post creation, viewing, and resolution features function correctly

The application is now protected against this class of attack and follows SQL injection prevention best practices.
