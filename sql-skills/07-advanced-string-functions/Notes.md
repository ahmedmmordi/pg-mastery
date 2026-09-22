# Advanced String Functions / Regex

## 1. Pattern matching: `LIKE` / `ILIKE`

The most common way to filter text without full regex.

```sql
SELECT name FROM Employees WHERE name LIKE 'A%';    -- starts with A (case-sensitive)
SELECT name FROM Employees WHERE name ILIKE 'a%';    -- starts with a or A (case-insensitive)
```

| Wildcard | Meaning |
|---|---|
| `%` | zero or more characters |
| `_` | exactly one character |

- `LIKE` is case-sensitive; `ILIKE` (PostgreSQL-specific) isn't.
- **Escaping a literal `%` or `_`**: use `ESCAPE` to define an escape
  character, then prefix the wildcard with it.

```sql
SELECT * FROM Products WHERE discount_code LIKE '50\%' ESCAPE '\';
-- matches the literal string "50%", not "50" followed by anything
```

---

## 2. Regex matching: `~`, `~*`, `!~`, `!~*`, `SIMILAR TO`

For anything `LIKE` can't express — character classes, alternation,
repetition.

```sql
SELECT name FROM Employees WHERE name ~ '^[A-M]';      -- matches (case-sensitive) regex
SELECT name FROM Employees WHERE name ~* '^[a-m]';      -- case-insensitive
SELECT name FROM Employees WHERE name !~ '[0-9]';       -- does NOT match
SELECT name FROM Employees WHERE name !~* 'error';      -- NOT match, case-insensitive
```

`SIMILAR TO` sits between `LIKE` and full regex — SQL-standard, supports
`%`/`_` plus a few regex features (`|`, `*`, `+`), but is more limited and
generally less useful than just using `~` directly:

```sql
SELECT name FROM Employees WHERE name SIMILAR TO '(A|B)%';  -- starts with A or B
```

**Tricky part — `~` doesn't anchor to the whole string by default.**
`name ~ 'cat'` matches `'cat'`, `'concatenate'`, and `'scatter'` — it
matches *anywhere* in the string, like `LIKE '%cat%'`. Use `^` and `$` to
anchor to the full string if that's the intent:

```sql
SELECT * FROM Words WHERE word ~ '^cat$';   -- exactly "cat", nothing more
```

---

## 3. Extracting and replacing with regex: `REGEXP_MATCHES`, `REGEXP_REPLACE`

```sql
-- Extract: returns an array of captured groups (or the whole match if no groups)
SELECT REGEXP_MATCHES('order-4521', '\d+');           -- {4521}
SELECT REGEXP_MATCHES('2024-01-15', '(\d+)-(\d+)-(\d+)'); -- {2024,01,15}

-- Replace: swap matched text for something else
SELECT REGEXP_REPLACE('hello world', 'o', '0');         -- "hell0 world" (first match only)
SELECT REGEXP_REPLACE('hello world', 'o', '0', 'g');     -- "hell0 w0rld" (all matches, 'g' flag)
```

**Tricky part — `REGEXP_MATCHES` without `'g'` is a set-returning function,
not a plain value.** Used in a plain `SELECT` list, it returns one row per
match if there are multiple matches in the string (or errors/behaves
unexpectedly if used carelessly alongside other columns); with the `'g'`
flag it explicitly returns **all** matches, one row each. For extracting
just one match/group as a normal scalar value, `SUBSTRING(str FROM
pattern)` (section 5) is often simpler and avoids the set-returning
behavior entirely.

```sql
SELECT REGEXP_REPLACE('2024-01-15', '(\d+)-(\d+)-(\d+)', '\3/\2/\1');
-- "15/01/2024" — \1, \2, \3 reference captured groups in the replacement
```

---

## 4. Finding a substring's position: `POSITION`

```sql
SELECT POSITION('world' IN 'hello world');   -- 7 (1-indexed)
SELECT POSITION('xyz' IN 'hello world');     -- 0, not NULL, if not found
```

**Tricky part**: returns `0` — not `NULL` — when the substring isn't
found. Checking `POSITION(...) IS NULL` to detect "not found" is a bug;
check `= 0` instead.

---

## 5. Extracting substrings: `SUBSTRING`, `LEFT`, `RIGHT`, `SPLIT_PART`

```sql
SELECT SUBSTRING('hello world', 1, 5);        -- "hello" (start, length)
SELECT SUBSTRING('hello world' FROM 7);        -- "world" (SQL-standard FROM/FOR syntax)
SELECT SUBSTRING('hello world' FROM 1 FOR 5);  -- "hello"

SELECT LEFT('hello world', 5);   -- "hello" — first 5 characters
SELECT RIGHT('hello world', 5);  -- "world" — last 5 characters
```

`SUBSTRING` also has a regex form, extracting the first match directly as
a scalar (no array, unlike `REGEXP_MATCHES`):

```sql
SELECT SUBSTRING('order-4521' FROM '\d+');  -- "4521"
```

**`SPLIT_PART`** — split a string on a delimiter, return one piece by
position:

```sql
SELECT SPLIT_PART('a,b,c', ',', 2);   -- "b"
SELECT SPLIT_PART('user@example.com', '@', 1);  -- "user"
```

**Tricky part**: 1-indexed, like `POSITION` and `SUBSTRING` — there is no
0th part. Asking for a part number beyond how many pieces exist returns an
empty string `''`, not `NULL` and not an error.

---

## 6. Modifying strings: `REPLACE`, `TRANSLATE`, `TRIM`, `LPAD`/`RPAD`, `REPEAT`, `REVERSE`

```sql
SELECT REPLACE('hello world', 'world', 'there');  -- "hello there" — substring replace
SELECT TRANSLATE('hello', 'el', 'ip');             -- "hippo" — character-by-character map
```

**`REPLACE` vs `TRANSLATE` — easy to mix up:**
- `REPLACE(str, substring, replacement)` swaps one exact **substring** for
  another string (they can be different lengths).
- `TRANSLATE(str, from_chars, to_chars)` maps **individual characters**
  one-to-one — `TRANSLATE('hello', 'el', 'ip')` replaces every `e` with
  `i` and every `l` with `p`, giving `'hippo'`. Useful for things like
  stripping/mapping a fixed character set at once; overkill (and the wrong
  tool) for substring swaps.

```sql
SELECT TRIM('  hello  ');                    -- "hello" — strips whitespace both sides by default
SELECT TRIM(LEADING '0' FROM '00042');        -- "42"
SELECT TRIM(TRAILING '.' FROM 'file.txt.');   -- "file.txt"
SELECT TRIM(BOTH 'x' FROM 'xxhelloxx');       -- "hello"

SELECT LPAD('42', 5, '0');   -- "00042" — pad on the left to a fixed length
SELECT RPAD('42', 5, '.');   -- "42..." — pad on the right

SELECT REPEAT('ab', 3);      -- "ababab"
SELECT REVERSE('hello');     -- "olleh"
```

`LPAD`/`RPAD` truncate instead of pad if the string is already longer than
the target length — worth knowing since it's not just "pad," it's "force
to exactly this length."

---

## 7. Combining strings: `CONCAT`, `CONCAT_WS`, `||`

```sql
SELECT first_name || ' ' || last_name AS full_name FROM Employees;
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM Employees;
SELECT CONCAT_WS(', ', city, state, zip) AS address FROM Locations;  -- WS = "with separator"
```

**Tricky part**: `||` returns `NULL` if *any* operand is `NULL` — one
missing piece silently blanks the whole concatenation. `CONCAT()` and
`CONCAT_WS()` instead treat `NULL` arguments as empty strings and skip
them, which is usually what's actually wanted (e.g. a middle name that's
sometimes missing shouldn't blank out the whole name).

---

## 8. Case conversion: `LOWER`, `UPPER`, `INITCAP`

```sql
SELECT LOWER('Hello World');    -- "hello world"
SELECT UPPER('Hello World');    -- "HELLO WORLD"
SELECT INITCAP('hello world');  -- "Hello World" — capitalizes the first letter of each word
```

Used constantly for case-insensitive comparisons when `ILIKE` isn't quite
the right tool (e.g. comparing two full column values for equality
regardless of case, not pattern-matching):

```sql
SELECT * FROM Users WHERE LOWER(email) = LOWER('Someone@Example.com');
```

**Tricky part**: wrapping a column in `LOWER()`/`UPPER()` inside a `WHERE`
makes the predicate non-sargable — same issue as `LENGTH(content) > 15`
from the Aggregate notes. A plain index on `email` can't be used here; the
real-world fix is an expression index (`CREATE INDEX ON Users
(LOWER(email))`) or a case-insensitive column type, not a rewrite of the
query itself.

`INITCAP` is naive about what counts as a "word boundary" — it capitalizes
after *any* non-letter character, so `INITCAP("o'brien")` gives
`"O'Brien"` (capitalizing after the apostrophe too), which isn't always
the desired result for names.

---

## 9. Length: `LENGTH` vs `CHAR_LENGTH`

```sql
SELECT LENGTH('hello');       -- 5
SELECT CHAR_LENGTH('hello');  -- 5
```

Equivalent for `text`/`varchar` in PostgreSQL — both count characters, not
bytes. `LENGTH()` is overloaded to also report byte length for `bytea`
(binary data), which is the main reason both names exist.

---

## 10. Formatting and conversion: `TO_CHAR`, `TO_NUMBER`, `FORMAT`

```sql
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD');       -- date formatting, e.g. "2024-01-15"
SELECT TO_CHAR(1234.5, 'FM999,999.00');    -- number formatting, "1,234.50"
SELECT TO_NUMBER('1,234.50', '999,999.00'); -- parses formatted text back to a number: 1234.50

SELECT FORMAT('Hello, %s! You have %s new messages.', 'Ali', 5);
-- "Hello, Ali! You have 5 new messages." — printf-style template
```

`TO_CHAR`'s format string vocabulary (`YYYY`, `MM`, `DD`, `HH24`, `MI`,
`SS`, `999`, `FM` to suppress padding, etc.) is worth having a reference
for — it's the standard way to turn dates/numbers into display strings in
PostgreSQL, distinct from casting (`::text`), which uses a fixed default
format instead of a customizable one.

---

## 11. Miscellaneous: `ASCII`, `CHR`, `MD5`

```sql
SELECT ASCII('A');    -- 65 — character to its code point
SELECT CHR(65);        -- "A" — code point back to a character (inverse of ASCII)

SELECT MD5('hello');  -- "5d41402abc4b2a76b9719d911017c592" — hash, fixed 32-char hex output
```

`ASCII`/`CHR` are the pair used for character-code manipulation (e.g.
"shift every letter by one position," Caesar-cipher-style problems).
`MD5` is a one-way hash — same input always produces the same fixed-length
output, useful for de-duplication checks, anonymizing/masking a value
while keeping it comparable, or generating a stable fingerprint of a row —
but it's not a general string function like the others, and it's not
meant for password storage (not designed to be slow/salted the way real
password hashing needs to be).