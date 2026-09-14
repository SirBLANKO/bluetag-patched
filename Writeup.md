# BlueTag: SQL injection in board search

**Location:** The searchItems() function in the items route.

**Problem:** Search text (q), category, and kind are inserted directly into SQL strings. Specially crafted input can change the search's conditions.

**Impact:** A visitor can manipulate search results and bypass filters. The vulnerable search can also bypass the condition that excludes listings marked removed.

**Live Demo:** https://bluetag-patched-uzoh.onrender.com

## Steps to reproduce:

1. Open the running wesite.
2. Set the category and kind filters to "all."
3. Search for "phone" and it will produces no matches.
4. Search for "phone' OR 1=1 --" and observe that you can now see every listing.
5. Observe whether listings appear even though they do not contain the search phrase.

**Expected behavior:** The app searches for the supplied text and returns no matches.

**Observed behavior before the patch:**

![Before patch screenshot 1](https://github.com/SirBLANKO/bluetag-patched/blob/main/Screenshot%202026-09-13%20175803.png?raw=true)

![Before patch screenshot 2](https://github.com/SirBLANKO/bluetag-patched/blob/main/Screenshot%202026-09-13%20180003.png?raw=true)

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
  return db.prepare(sql).all();  // Unsafe: user input was interpolated into the SQL above.
}
```

In this vulnerable version, user input is directly interpolated into the SQL string using template literals. An attacker can submit `phone' OR 1=1 --` as the search term. The apostrophe closes the string and allows them to inject SQL logic.

The injected search becomes:
```sql
WHERE items.status != 'removed' AND items.title || ' ' || items.description || ' ' || items.location LIKE '%phone' OR 1=1 --%'
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

The patched version uses `?` placeholders and passes values separately via `.all(...params)`. This ensures user input is treated as data, not SQL code. Even if an attacker enters `phone' OR 1=1 --`, it's treated as a literal string value to search for, not as executable SQL.

**Fix:** In the searchItems() function, replaced all interpolated values with `?` placeholders for the search term (q), category, and kind parameters. Passed their values separately through the `.all(...)` method.

**Observed behavior after the patch:**

![After patch screenshot 1](https://github.com/SirBLANKO/bluetag-patched/blob/main/Screenshot%202026-09-13%20180303.png?raw=true)

![After patch screenshot 2](https://github.com/SirBLANKO/bluetag-patched/blob/main/Screenshot%202026-09-13%20180351.png?raw=true)

After the patch, entering SQL injection payloads into the search bar no longer affects the results. The SQL injection was successfully prevented, and the search function now behaves normally, returning only actual matches.

## Exact test requests

The following test requests are documented for reproducibility. Category and kind are tested by changing URL parameters directly, because their dropdown menus do not offer these values.

**Baseline (expected: no matches):**
```
https://bluetag-patched-uzoh.onrender.com/?q=phone&category=all&kind=all
```

**Search field injection (expected: no matches and no SQL error):**
```
https://bluetag-patched-uzoh.onrender.com/?q=phone%27%20OR%201%3D1%20--&category=all&kind=all
```

**Category field injection (expected: no matches and no SQL error):**
```
https://bluetag-patched-uzoh.onrender.com/?category=keys%27%20OR%201%3D1%20--&kind=all
```

**Kind field injection (expected: no matches and no SQL error):**
```
https://bluetag-patched-uzoh.onrender.com/?category=all&kind=lost%27%20OR%201%3D1%20--
```

## Normal functionality checks:

| Test | Actual result |
|---|---|
| Baseline search | No matching listings and no search error. |
| Search field injection | No matching listings and no search error. |
| Category field injection | No matching listings and no search error. |
| Kind field injection | No matching listings and no search error. |
| Search for a known item | The keys listing appeared; the headphones listing did not. |
| Category and kind filters together | Both filters applied; the two filter combinations returned the expected results. |
| Search containing an apostrophe | The O'Brien listing appeared without a search error. |
| Register, sign in, and sign out | Registration worked; signing out blocked protected-page access; signing in restored access. |
| Create and view a post | Both listings displayed the correct information and remained available after refreshing. |
| Resolve your own post | The selected listing became resolved; the other listing's status remained unchanged. |

## Summary

This SQL injection vulnerability in the BlueTag board search function has been patched. The fix converts the dynamic SQL search construction to use prepared statements with parameterized searches.

**Key improvements:**
- User input is no longer directly interpolated into SQL strings using template literals
- All three vulnerable parameters (q, category, kind) now use `?` placeholders
- Values are passed as separate parameters to the database driver, ensuring they're treated as data only
- The patch maintains full functionality while eliminating the SQL injection vulnerability

**Verification status:**

All ten manual checks performed on the patched local application matched their expected results. The tests verified that search, category, and kind injection fixes function correctly with the patch in place.

The patch addresses the identified SQL injection in searchItems() by binding the search, category, and kind values separately from the SQL command. The normal-use tests performed continued to pass, confirming that the patch doesn't break existing functionality.
