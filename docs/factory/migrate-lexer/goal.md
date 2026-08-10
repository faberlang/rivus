# Goal: migrate-lexer

**Status**: active — implementation landed 2026-08-10 (check green); residuals: LexErrorKind mirror swap, keyword-row compaction
**Created**: 2026-08-10
**Target repo**: /Users/ianzepp/work/faberlang/rivus
**Factory artifact dir**: docs/factory/migrate-lexer/
**Primary surface**: `crates/radix-lexer/src/{lib,token,keywords,scan,cursor}.rs` → `rivus/src/lexer/{token,keywords,scan,cursor,interner}.fab`
**Depends on**: migrate-diagnostics
**Related**: [docs/CAMPAIGN.md](../../CAMPAIGN.md), [docs/STRUCTURE.md](../../STRUCTURE.md)

## Summary

Ports the `radix-lexer` crate — the lexical front end (token vocabulary, keyword
registry, scanner state machine, UTF-8 cursor, and the `Interner`/`Symbol`
foundation) — into `rivus/src/lexer/` as the en-surface Faber mirror. It
produces the token stream, lexical diagnostics, and interned symbols that every
later stage consumes; per CAMPAIGN L4 (source-text firewall), nothing after lex
re-reads source bytes. The interpreter needs the full token/error surface; the
reader-locale pack machinery (KeywordPack, fallbacks) and the serialization
wire contract are dropped (L3: en surface; no artifact loading).

## Crate analysis

`radix-lexer` (v0.80.0) is a foundation crate re-exported as `radix::lexer`.
Production source: 3,287 lines (cursor 136, lib 224, token 496, scan 1040,
keywords 1391) — matches the ~3.3k target in STRUCTURE.md.

- **Data flow**: `lex(source)` (lib.rs:62) → `Lexer::lex()` (scan.rs:239) →
  `scan_token()` dispatch → token emission into `tokens: Vec<Token>`, errors
  into `errors: Vec<LexError>`; always terminates with `TokenKind::Eof`. Output
  is a `LexResult` bundling tokens, errors, locale metadata, and the `Interner`.
- **Core types**: `TokenKind` (~180 variants, 8 with payload: Ident/Underscore/
  Integer/Float/String/AsciiString/BacktickString/OctetiString/LineComment —
  token.rs:76-328); `Token { kind, span }`; `Symbol(u32)` — a handle into the
  `Interner`'s string table, local to one lexer run; `Span { start, end: u32 }`
  re-exported from `radix-diagnostics` (byte offsets, not char counts).
- **Scanner** (scan.rs): single-pass, error-resilient. Sub-scanners: line
  breaks/`\r\n`, operators (incl. `!`/`?` lookahead compounds), comments
  (`#` line-start only; inline `#` after code → `InlineComment` error;
  `//`/`/* */` → `InvalidCommentSyntax`), five string families
  (`"…"`, `'…'`, `` `…` ``, `|…|`, `«…»`), numbers (dec/hex/oct/bin, `_`
  separators, exponents, F-056 trailing-garbage, F-057 underflow), identifiers
  (XID + `_`, NFKC-normalized for symbols). `§` → `RemovedSectionDirective`.
- **Keyword policy** (keywords.rs): keywords are **never emitted** — every
  keyword lexes as `Ident`; the registry maps spelling → locale-independent
  identity `TokenKind`, and the parser claims keywords by identity. The
  Latin-path lexer records `reader_keyword_latin_keys` in `LexResult` (consumed
  by radix-parser lib.rs:405/580 — a parser contract; keep).
- **Interner** (scan.rs:60-183): `map: FxHashMap<String, Symbol>` +
  `strings: Vec<String>`. Two intern paths: `intern` (NFKC, identifiers /
  keyword lookup) and `intern_raw` (source bytes, literals/comments — PRP-013).
- **Dependencies**: `radix-diagnostics` (Span, LexErrorKind, DiagnosticArg —
  all migrate-diagnostics scope), `rustc-hash`, `serde` (wire contract only),
  `unicode-ident` (XID tables), `unicode-normalization` (NFKC tables). The last
  two have no Faber equivalent — see frictions.
- **Not present**: `unsigned_integer.rs` does not exist in the live crate — the
  u64 handling lives in `scan_number`/`scan_radix_int` (scan.rs:390-425,
  807-903); only `unsigned_integer_test.rs` (test-only) is included from scan.rs.

## Module mapping (crate → rivus)

| Radix file | Rivus file | Faber representation / en identifiers |
|---|---|---|
| `token.rs` | `lexer/token.fab` | `TokenKind` → `union` (variants port 1:1; payload variants carry `Symbol`/`num`; `Integer`/`Float` → `num`). `Token` → `class Token { kind, span }`. `Symbol` → `class Symbol { id: num }` (arena-ID: index into interner `strings`). `is_comment`/`is_trivia` → fns; `is_keyword` → registry-driven fn (see frictions). Imports `Span` from `rivus:diagnostics/span`. |
| `keywords.rs` | `lexer/keywords.fab` | `KeywordSpec` → class (text, `token_kind: TokenKind?`, scope, category). `KeywordScope` → union (`Contextual { owners: lista<KeywordOwner> }`, `Alias { canonical: textus }`). `KeywordOwner` / `KeywordCategory` → union of unit variants. `KeywordGroupSpec` → class. `KEYWORD_SPECS` / `KEYWORD_GROUP_SPECS` → const lists; lookup fns ported. |
| `scan.rs` (Interner) | `lexer/interner.fab` | `Interner` → class { `strings: lista<textus>`, `map: tabula<textus, Symbol>` } — ordered tabula replaces FxHashMap (determinism, CAMPAIGN metric 3). `intern` (NFKC) / `intern_raw` (bytes) / `resolve` / `strings`. |
| `scan.rs` (Lexer + scanner) | `lexer/scan.fab` | `Lexer` → class (cursor, interner, `tokens: lista<Token>`, `errors: lista<LexError>`, `reader_keyword_latin_keys: tabula<textus,textus>`, `reader_type_latin_keys`, `current_line`, `line_has_code`). Scanner dispatch + all sub-scanners → fns on Lexer. `LexResult` → class. `LexError` → class { kind: LexErrorKind (from diagnostics), span, args }. `INLINE_COMMENT_ISSUE` → const. |
| `cursor.rs` | `lexer/cursor.fab` | `Cursor` → class over owned `textus` (no lifetimes): { source, pos }, `peek`/`peek_next` → `textus?` (one scalar), `advance`/`eat`/`eat_while`/`slice`/`rest`. `eat_while` takes a predicate fn on one-scalar textus. |
| `lib.rs` | (folded into scan.fab; no lib.fab) | `lex()` top-level fn; `LexResult::success()`. `lex_with_locale_pack`, `KeywordPack`, `LatinKeywordFallback`, `LocaleFallback`, `LocaleSuggestion` dropped. |

## Port notes / frictions

- **No `char`/`ord`** (cursor.fab, scan.fab): single characters become one-scalar
  `textus`; comparisons are textus equality (`c == "."`); digit/alpha/hex
  checks become local fns on one-scalar textus. `is_control` in
  `diagnostic_char` (scan.rs:998) → ASCII approximation (U+0000–U+001F, U+007F).
  `is_bidi_control` → fixed range check, ports directly.
- **Byte-offset spans vs textus offsets** (decision, default): `Span` must stay
  byte offsets (radix diagnostics contract). Cursor `pos` advances by the
  equivalent of `len_utf8`. Verify at implementation whether textus exposes
  byte offsets; if it counts scalars, the cursor must track bytes separately
  for span parity.
- **NFKC + XID tables** (decision, default): `unicode-normalization` and
  `unicode-ident` are table-driven crates with no Faber equivalent. Default:
  ASCII subset — identifier boundaries `[A-Za-z_][A-Za-z0-9_]*` and NFKC as
  identity for ASCII — which is exact for the Latin corpus (L3), plus explicit
  handling of non-ASCII identifier chars (accept via a small XID table or
  reject — reject changes must match the stepper's accept/reject). Flag as a
  deviation if widened later.
- **Numeric parsing** (scan.fab): `u64::from_str_radix` (hex/oct/bin) and
  `parse::<u64>/<f64>` have no direct stdlib equivalent — implement
  digit-by-digit accumulation with overflow → `InvalidNumber`, and float
  overflow (`is_infinite`) / underflow (F-057 `has_non_zero_significand`)
  checks on the parsed `num`. Port `clean_numeric_text` (`_` stripping) as-is.
- **Registry lookups**: `keyword_by_text`/`keyword_by_kind` (OnceLock +
  `mem::Discriminant` hashing, keywords.rs:1244-1276) are Rust optimizations —
  in Faber, linear scan over `KEYWORD_SPECS` (~180 rows, comparable union
  match) is fine; no discriminant hash needed. `TokenKind::is_keyword`
  (~250-line `matches!`) → iterate the registry's `token_kind` values or a
  big `match`; prefer registry-driven.
- **Static data**: `&'static [KeywordSpec]` / `Contextual(&'static [KeywordOwner])`
  → const `lista` values; validate via `validate_keyword_groups` port (0 errors).
- **Folding lib.rs**: STRUCTURE.md fixes the lexer file list (no lib.fab);
  crate-level `lex()` and `LexResult` fold into scan.fab. `LocaleFallback` /
  `LocaleSuggestion` production is dead on the en surface (only pack path) —
  drop fields and keep `LexResult` lean (tokens, errors, latin keys, interner).
- **Naming**: Radix identifiers port 1:1 (L3); variant names (`Discretio`,
  `Scribe`, …) and registry `text` spellings (`"functio"`, …) unchanged. The
  en-surface keyword set (fn/class/union/enum/…) is the *language* Rivus is
  written in — it is unrelated to the Latin Faber keyword spellings being lexed
  for user programs.

## Done-when

1. `lexer/{token,keywords,scan,cursor,interner}.fab` exist; package builds green
   on the en surface (`faber check` on touched files, one closeout run per
   workspace rules).
2. `lex()` on a trivial program emits the expected stream (Ident/Int/String/
   Newline/Eof) with a terminal `Eof` always, even after errors.
3. Number parity: dec/hex/oct/bin, `_` separators, floats/exponents; overflow,
   underflow (F-057), and F-056 trailing-garbage produce `InvalidNumber` +
   `TokenKind::Error` with the exact issue facts.
4. String family parity: `"`, `'`, `` ` ``, `|…|`, `«…»` interned **raw** (no
   NFKC); unterminated forms produce `UnterminatedString` with the right issue.
5. Comment parity: line-start `#` → `LineComment`; inline `#` after code →
   `InlineComment` + `Error`; `//` and `/* */` → `InvalidCommentSyntax` +
   `Error`; `§` → `RemovedSectionDirective`.
6. `Interner`: `intern` vs `intern_raw` paths, `resolve` round-trip; symbols
   are local to one `LexResult`; interner is deterministic (ordered `tabula`).
7. Keyword registry: all ~180 `KEYWORD_SPECS` rows and 5 groups ported;
   `lookup_keyword_spec`, `keyword_token_kind`, `keyword_spelling_for_token`,
   `keyword_allowed_as_ident`, `validate_keyword_groups` behave as radix
   (groups validate with 0 errors).
8. Reject parity: lexical errors map to the same `LexErrorKind` +
   `issue` facts as radix; spans are byte offsets into the original source.
9. `LexResult` carries `reader_keyword_latin_keys` (parser contract) +
   `interner` + `errors`; `success()` ⇔ empty errors.

## Out of scope

- Reader-locale packs (`KeywordPack` trait, `lex_with_locale_pack`,
  `LatinKeywordFallback`, `LocaleFallback`/`LocaleSuggestion`, `type_latin_key`)
  — L3 en surface; pack spellings are a separate future goal if ever needed.
- Serde wire contract (`to_string_table`, `from_strings`) — interpreter loads
  no artifacts; `resolve`/`strings` is the stepper's symbol surface.
- Parsing, diagnostics rendering, and `LexErrorKind`/`Span`/`DiagnosticArg`
  definitions (migrate-diagnostics owns the diagnostics module).
- Porting radix `*_test.rs` fixtures wholesale (they are the oracle inventory
  for behavior parity, not source to copy); new Faber tests are strongly
  preferred per repo protocol.
