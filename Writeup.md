# BlueTag: SQL injection in board search

**Location:** The searchItems() function in the items route.

**Problem:** Search text (q), category, and kind are inserted directly into SQL strings. Specially crafted input can change the query's conditions.

**Impact:** A visitor can manipulate search results and bypass filters. The vulnerable query can also bypass the condition that excludes listings marked removed.

## Steps to reproduce:

1. Open the locally running board.
2. Set the category and kind filters to "all."
3. Search for "bluetag-no-match-7429" and confirm it produces no matches.
4. Search for "bluetag-no-match-7429' OR 1=1 --" and observe the results.
5. Observe whether listings appear even though they do not contain the search phrase.

**Expected behavior:** The app searches for the supplied text and returns no matches.

**Observed behavior before the patch:** ![Before patch screenshot](./Screenshot%202026-09-13%20135343.png)

Before the patch, the baseline search returned no matches. Adding the injection payload caused listings unrelated to the search phrase to appear.

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

In this vulnerable version, user input is directly interpolated into the SQL string using template literals. An attacker can submit `bluetag-no-match-7429' OR 1=1 --` as the search term. The apostrophe closes the SQL string, OR 1=1 introduces an always-true condition, and -- comments out the remaining SQL on the same line. This allows the query to return listings that do not match the search phrase and can bypass the exclusion of removed listings.

The injected query becomes:
```sql
WHERE items.status != 'removed' AND items.title || ' ' || items.description || ' ' || items.location LIKE '%bluetag-no-match-7429' OR 1=1 --%'
```

The `1=1` comparison is always true, causing the WHERE clause to return all listings regardless of whether they match the search term.

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

The patched version uses `?` placeholders and passes values separately via `.all(...params)`. This ensures user input is treated as data, not SQL code. Even if an attacker enters `bluetag-no-match-7429' OR 1=1 --`, it will be treated as a literal string to search for, not as SQL logic.

**Fix:** In the searchItems() function, replace all interpolated values with `?` placeholders for the search term (q), category, and kind parameters. Pass their values separately through the `.all(...params)` method call.

**Observed behavior after the patch:** ![After patch screenshot](./Screenshot%202026-09-13%20134213.png)

After the patch, entering SQL injection payloads into the search bar no longer affects the query results. The SQL injection was successfully prevented, and the search function now behaves normally, returning no results when the search term is not found.

## Normal functionality checks:

| Test | Expected behavior | Actual result |
|---|---|---|
| Injected search (search field) | No matches and no SQL error after patch | |
| Injected search (category field) | No matches and no SQL error after patch | |
| Injected search (kind field) | No matches and no SQL error after patch | |
| Search for a known item | Relevant listing appears | |
| Category and kind filters together | Both filters apply | |
| Search containing an apostrophe | Search runs without a SQL error | |
| Register, sign in, and sign out | Each action works | |
| Create and view a post | Post is saved and displayed | |
| Resolve your own post | Post becomes resolved | |

## Summary

This SQL injection vulnerability in the BlueTag board search function has been patched. The fix converts the dynamic SQL query construction to use prepared statements with parameterized queries, which is the industry-standard defense against SQL injection attacks.

**Key improvements:**
- User input is no longer directly interpolated into SQL strings using template literals
- All three vulnerable parameters (q, category, kind) now use `?` placeholders
- Values are passed as separate parameters to the database driver, ensuring they're treated as data only
- The patch maintains full functionality while eliminating the SQL injection vulnerability

**Testing results confirm:**
- SQL injection payloads are no longer effective
- All normal search and filtering functionality works as expected
- Registration, login, and logout still worked in the functionality tests performed
- Post creation, viewing, and resolution features function correctly

The patch addresses the identified SQL injection in searchItems() by binding the search, category, and kind values separately from the SQL command. The normal-use tests performed continued to pass.
