# BlueTag: SQL injection in board search

**Location:** The searchItems() function in the items route.

**Problem:** Search text (q), category, and kind are inserted directly into SQL strings. Specially crafted input can change the query's conditions.

**Impact:** A visitor can manipulate search results and bypass filters. The vulnerable query can also bypass the condition that excludes listings marked removed.

## Steps to reproduce:

1. Open the locally running board.
2. Set the category and kind filters to "all."
3. Search for something not on the board and confirm it produces no matches.
4. Search for that same thing but add `OR 1=1 --` to the end of it.
5. Now you can see things everything, even items that were already resolved.

**Expected behavior:** The app searches for the supplied text and returns no matches.

**Observed behavior before the patch:** ![Before patch screenshot](./Screenshot%202026-09-13%20134213.png)

Before the patch, putting any command into the serch bar cause the page to return all matches, even matches that we're already taken down from the board.

**Cause:** User input becomes part of the SQL command before the command is prepared.

**Fix:** Replace interpolated values with ? placeholders for all three inputs and pass their values separately through .all(...params).

**Observed behavior after the patch:** ![After patch screenshot](./Screenshot%202026-09-13%20135343.png)

After the patch, putting any command into the search bar caused the page to return no valid matches.

## Normal functionality checks:

| Test | Expected behavior | Actual result |
|---|---|---|
| Injected search | No matches or SQL error after patch | [Fill in] |
| Search for a known item | Relevant listing appears | [Fill in] |
| Category and kind filters together | Both filters apply | [Fill in] |
| Search containing an apostrophe | Search runs without a SQL error | [Fill in] |
| Register, sign in, and sign out | Each action works | [Fill in] |
| Create and view a post | Post is saved and displayed | [Fill in] |
| Resolve your own post | Post becomes resolved | [Fill in] |
