# SCL:V1 — OFFICIAL GRAMMAR SPECIFICATION  
### FINAL • FROZEN • UNCHANGING • PROTOCOL-GUARANTEED

This document defines the **only valid grammar** for SCL:V1.  
All engines and SDKs MUST match this **exactly**: bytes in → AST out → canonical JSON out → hash out.

Canonical source of truth: SCL:V1 Specification (this document)  
Reference implementation: C (scl-core)  
Conformance tools: Python, C++, Rust (MUST match this spec and scl-fixtures)


**This spec is normative.** Any behavior not explicitly allowed here is forbidden.

No changes will ever be made to SCL:V1 after launch. Future revisions belong to **SCL:V2**.

---

# 0. NORMATIVE DEFINITIONS (MANDATORY)

## 0.1 Byte model
SCL:V1 documents are sequences of bytes.

## 0.2 Encoding (UTF-8 only)
- The entire document MUST be valid UTF-8 (RFC 3629).
- Engines MUST NOT apply Unicode normalization (no NFC/NFD/NFKC/NFKD). **Bytes are canonical.**
- Any invalid UTF-8 sequence MUST raise **E001**.
- For **E001 invalid UTF-8**, `byte_offset` MUST be the first byte of the invalid UTF-8 sequence.

## 0.3 Line endings (LF only)
- The only permitted line ending is LF (`0x0A`).
- CR (`0x0D`) is forbidden **anywhere** in the document (including inside quoted strings and raw content) and MUST raise **E001** at that byte offset.

## 0.4 Forbidden bytes (global)
- Tabs (`0x09`) are forbidden **anywhere** in the document (including inside quoted strings and raw content) and MUST raise **E001** at that byte offset.

## 0.5 Forbidden control characters inside quoted strings
Inside any quoted string (handle tags or SCL quoted content):
- Any Unicode code point U+0000 through U+001F OR U+007F is forbidden.
- A literal LF inside a quoted string is forbidden (quoted strings never span physical lines).
Violation MUST raise **E001**.

Normative note: control-character checks apply to decoded Unicode code points **after** UTF-8 validation (not raw bytes).

## 0.6 Deterministic error rule (“first failure”)
- Parsing proceeds left-to-right, top-to-bottom.
- The engine MUST report the **first error by byte offset** (the lowest offset where parsing cannot continue).
- `byte_offset` is **0-based**, counting from the first byte of the document.
- If multiple rules fail at the same byte offset, precedence is:
  1) **E001** (encoding/control/CR/tab)
  2) Header/block structural errors (E101/E102/E103/E104/E105)
  3) Handle/tag errors (E201/E202)
  4) **E900** (only if no other code applies)

Recommendation (for parity): engines SHOULD expose `(code, byte_offset)` internally even if the public API only returns `code`.

## 0.7 Token bytes and comparisons
Unless explicitly stated otherwise:
- All fixed keywords and punctuation are ASCII bytes and MUST match exactly.
- “No trailing spaces” means **no `0x20`** bytes before LF/EOF where disallowed.

---

# 1. DOCUMENT STRUCTURE (TOP LEVEL)

A valid SCL:V1 document MUST be exactly:

1) Header line: `SCL:V1`  
2) One blank line (exactly `\n\n`)  
3) Handles block  
4) Immediately followed by SCL block **with no blank line between**  
5) End of document exactly at the final `}` of the SCL block (no extra bytes)

**No other blank lines are permitted anywhere else in the document.**  
(A “blank line” is a physical line consisting of exactly LF with no preceding bytes.)

**Top-level order is fixed:** `Header → Handles → SCL`.

Any violation MUST raise a deterministic error per §0.6 and the error codes table.

---

# 2. HEADER

Valid header line is exactly:

```
SCL:V1
```

Rules:
- The only valid version header for SCL:V1 documents is exactly `SCL:V1`.
- Engines MUST reject any other version identifier with E101.
- Future versions (e.g. SCL:V2) require separate explicit support and are not valid input to V1 engines.
- First byte of file MUST be `S` (`0x53`).
- No UTF-8 BOM is allowed. If the file begins with `0xEF 0xBB 0xBF`, raise **E101** at byte offset 0.
- No leading or trailing spaces.
- Followed by exactly one blank line (`\n\n`) and no additional blank lines.


Invalid header → **E101** (byte offset is the first byte where the header cannot match, per §0.6).

---

# 3. HANDLES BLOCK

## 3.1 Exact block delimiters
The handles block MUST be:

- Opening line exactly: `handles {` followed immediately by LF (`\n`).
  - No trailing spaces, nothing after `{`.
- Closing line exactly: `}` followed by LF (`\n`).

**Missing handles block → E102**  
**EOF before handles closes → E103**

## 3.2 Handles block contents
- At least one handle definition is required. An empty handles block (`handles {\n}\n`) MUST raise **E102** at the byte offset of the `}` that closes the block.
- No nested blocks.
- Inside `handles { ... }`, every physical line (each ending in LF) MUST be either:
  - a handle definition line (§3.3), OR
  - the closing delimiter line `}\n` (§3.1).
- Indentation, if present on handle definition lines, MUST be spaces only (`0x20`).
- Whitespace-only lines are forbidden: any line whose bytes before LF are zero or more spaces only (matches `^[ ]*\n$`) is forbidden inside the handles block and MUST raise **E102** at the byte offset of the first byte of that line.
  - For an empty physical line (`\n`), the “first byte of that line” is the LF byte.
- Any other non-empty line inside the handles block that is not a valid handle definition line (§3.3) and not `}\n` MUST raise **E102** at the byte offset of the first byte of that line, unless a more specific code applies (**E001**, **E201**, **E202**).

---

## 3.3 HANDLE DEFINITION LINE GRAMMAR (EXACT)

A handle definition is one physical line ending with LF:

```
<indent><handle_id>(<tag_list>)\n
```

Where:

### <indent>
- Zero or more spaces (`0x20`).

### <handle_id>
Regex (ASCII only):
```
^[A-Za-z_][A-Za-z0-9_]*$
```
Invalid → **E201**

### Exact punctuation / spacing rules
To eliminate interpretation:
- No spaces are allowed between `<handle_id>` and `(`.
- No spaces are allowed after `(` or before `)`.
- No spaces are allowed around commas in tag lists.
- No trailing spaces are allowed at end of line.

### Deterministic handle-line parsing and error mapping (MANDATORY)
Engines MUST parse each non-empty handles-block line using the following fixed procedure, and MUST emit the specified error code when the first failing condition is encountered (per §0.6):

1) Parse `<indent>` as zero or more bytes `0x20`.
2) Parse `<handle_id>` bytes: immediately after `<indent>`, the next byte MUST begin a handle_id (matching `[A-Za-z_]`). Consume bytes up to but not including the next `(` byte (or LF if no `(` occurs on the line). The consumed span MUST match the ASCII regex `^[A-Za-z_][A-Za-z0-9_]*$` exactly. Otherwise raise **E201** at the first byte that violates the regex (or at the current byte if the span is empty).
3) The next byte MUST be `(` with no intervening spaces.
   - If the next byte is LF (i.e., no `(` present on the line), raise **E201** at the LF byte.
   - If the next byte is a space `0x20`, raise **E201** at that space byte (spacing rule violation).
4) Parse `<tag_list>` inside the parentheses using §3.4. Any failure after consuming `(` MUST raise **E202** (unless **E001** applies).
5) After the closing `)`, the next byte MUST be LF (`0x0A`) with no trailing spaces. If any byte other than LF occurs after `)`, raise **E201** at the first such byte.

---

## 3.4 TAG LIST GRAMMAR (EXACT)

Tags appear inside parentheses:

```
id("tag1","tag2")
```

Rules:
- Tag list MUST contain one or more tags.
- Empty tag list `()` is forbidden → **E202**
- Tags MUST be double-quoted with `"` characters.
- Commas MUST separate tags; no trailing comma is allowed.
- No spaces are allowed anywhere inside the parentheses (including after commas).
- Each tag is a quoted string token that:
  - does not span lines,
  - contains no forbidden control characters (per §0.5),
  - is valid UTF-8 in bytes.

Invalid tag syntax → **E202**  
Invalid UTF-8/control/CR/tab → **E001**

Deterministic parsing rule for tags: once inside `(`, any token that is not `"` begins an error (**E202**) unless it is `)` (empty list → **E202**).

---

# 4. SCL BLOCK

## 4.1 Exact block delimiters
The SCL block MUST be:

- Opening line exactly: `scl {` followed immediately by LF (`\n`).
  - No trailing spaces, nothing after `{`.

- The document MUST end **immediately** after the closing `}` that terminates the SCL block with **no newline and no trailing bytes**.

**Missing SCL block → E104**  
**EOF before SCL block closes → E105**

If EOF occurs before a valid SCL terminator line is encountered, the error MUST be **E105**; otherwise, an invalid token where SCL content or a terminator is required MUST be **E104**.


---

# 4.2 CONTENT MODES (STRICT, EXACTLY ONE MODE)

Immediately after the line `scl {\n`, the content MUST be in exactly one of two modes:

Once a mode is selected by the first content line, all subsequent content lines MUST conform to that same mode; any deviation MUST raise **E104** (unless **E001** applies).


## MODE A — QUOTED MODE (ONE OR MORE QUOTED LINES)

A quoted-mode body is one or more physical lines, each of the form:

```
<indent>"<bytes>"\n
```

Rules:
- `<indent>` is zero or more spaces (`0x20`).
- Each quoted line MUST contain exactly one opening `"` and one closing `"` on the same physical line.
- No bytes are permitted after the closing `"` on that line (i.e., the byte after `"` MUST be LF).
- Bytes inside quotes are taken literally; no escape processing is performed in V1.
- Each quoted token MUST contain no forbidden control characters (per §0.5).

Termination:
- Quoted mode ends only when the next line is exactly `}` and the document ends immediately after that `}` (i.e., `}` is immediately followed by EOF with no trailing LF).

AST content construction:
- For each quoted line, strip outer quotes and take inner bytes.
- Join inner contents with `\n` between lines.

Invalid quoted line syntax → **E104** unless **E001** applies.

## MODE B — RAW MODE (MULTILINE, UNQUOTED)

Raw mode begins when the first content line after `scl {\n` does NOT begin (after optional spaces) with a double quote `"`.

Rules:
1) Raw content begins on the line after `scl {\n`.
2) Raw mode ends only when the **final** bytes of the document form one terminator line matching exactly:
   - optional spaces (`0x20`)*, then `}` (`0x7D`), then **EOF**
   - That is: `^[ ]*}$` followed immediately by EOF (no trailing LF).
3) Terminator whitespace is structural:
   - If a candidate terminator line matches `^[ ]*}[ ]+$` (i.e., `}` followed by spaces before LF/EOF), this MUST raise **E104** at the first trailing space byte after `}`.
4) Empty physical lines within raw content are allowed and are treated as content; the document-level “no other blank lines” restriction applies only outside the SCL block (i.e., it does not constrain raw-mode content).
5) Raw content must be valid UTF-8 → **E001**.

AST content construction:
- Raw content is the concatenation of all raw lines before the terminator line, joined with `\n`.
- The terminator line’s optional leading spaces are not part of content.

---

# 5. AST CANONICAL FORM (REQUIRED)

Every engine MUST emit exactly this AST object model:

```
{
  "type":"Document",
  "version":"SCL:V1",
  "handles":[
    {
      "type":"Handle",
      "id":"<id>",
      "tags":["<tag1>","<tag2>"]
    }
  ],
  "scl":{
    "type":"SclBlock",
    "content":"<content-bytes-as-utf8-string>",
    "refs":[],
    "hints":[]
  }
}
```

Invariants:
- `refs` and `hints` MUST exist and MUST be empty arrays in V1.
- `handles` MUST preserve input order.
- `tags` MUST preserve input order.
- `content` MUST equal the exact mode-derived content (§4.2).
- No extra fields. No nulls.

---

# 6. CANONICAL JSON AND DOCUMENT HASH (NORMATIVE)

SCL:V1 Document canonical JSON output uses only JSON objects, arrays, and strings.
Numbers, booleans, and null do not appear in the Document canonical JSON output.

Note: scl-fixtures/vectors_v1.json may include numbers/booleans as test metadata (e.g., input_len, expected_ok). Those are not part of the Document canonical JSON.


The canonical JSON bytes are the sole hash input for SCL:V1.

Engines MUST canonicalize the parsed document (AST canonical form; see §5) to canonical JSON bytes as defined in this section. The Document Hash is defined as:

doc_hash = SHA256(canonical_json_bytes)

Where canonical_json_bytes are the exact UTF-8 byte sequence of the canonical JSON output defined below.

# 6.1 Canonical JSON rules (normative)

Encoding
Canonical JSON MUST be UTF-8 encoded and MUST NOT include a UTF-8 BOM.

Single-line / no structural whitespace
Output MUST be exactly one line and MUST contain no whitespace bytes (0x20, 0x09, 0x0A, 0x0D) outside JSON strings.

Objects — key uniqueness
Object keys MUST be unique. Any document whose canonical form would produce an object with duplicate keys is invalid.

Objects — key ordering
Object keys MUST be sorted by binary lexicographic order of the UTF-8 byte sequences of the key strings (no locale collation; no normalization).

Compare byte-by-byte; if one key is a strict prefix of the other, the shorter key sorts first.

Arrays
Arrays preserve order.

Strings

Unicode validity: String values are sequences of Unicode scalar values. Lone UTF-16 surrogate code points MUST NOT appear in canonical JSON strings.

Escaping (minimal): Strings MUST be serialized using minimal escaping. The only permitted escapes are:
(a) " and \, which MUST be escaped, and
(b) control characters U+0000 through U+001F, which MUST be escaped using \u00XX.
No other escapes are permitted (e.g., \/, \b, \f, \n, \r, \t, or \uXXXX for non-control characters).


JSON syntax
No trailing commas.

Schema constraints
Only the fields defined in §5 may appear. No numbers, booleans, or null values are permitted.


# 6.2 Document Hash invariance (normative)

The hash input is the canonical JSON byte sequence exactly as emitted by the engine (no Unicode normalization, newline translation, or whitespace insertion/removal).

Any two SCL inputs that parse to the same AST canonical form (§5) MUST produce identical canonical JSON bytes (§6.1) and therefore MUST produce identical doc_hash. In particular, whitespace-equivalent / formatting-variant inputs MUST collide under doc_hash.

The Document Hash is not SHA256(raw_input_bytes). If an implementation exposes a raw-input checksum, it MUST be named distinctly (e.g., hash_raw_input) and MUST NOT be described as the Document Hash.



# 7. ERROR CODES (E-SERIES)

| Code  | Meaning |
|-------|---------|
| **E101** | Missing or invalid `SCL:V1` header |
| **E102** | Handles block missing, empty, or contains an invalid line where a handle definition or closing } is required |
| **E103** | Unclosed handles block (EOF before its closing `}`) |
| **E201** | Invalid handle ID |
| **E202** | Invalid tag list or tag token |
| **E104** | Missing SCL block, or invalid token where SCL content or SCL terminator line is required (excluding premature EOF) |
| **E105** | Unclosed SCL block (EOF before valid SCL termination) |
| **E001** | Invalid UTF-8, CR, tab, or forbidden control character |
| **E900** | Internal parser error |

---

# 8. FORBIDDEN SYNTAX
Everything not explicitly allowed by this spec is forbidden, including:
- Comments of any form
- Additional top-level blocks
- Nested blocks
- Alternative keywords/casing
- Extra blank lines (outside raw-mode content as described in §4.2)
- Trailing bytes after the final `}`

---

# 9. CHANGE CONTROL
SCL:V1 is frozen after launch. Any change requires SCL:V2.

