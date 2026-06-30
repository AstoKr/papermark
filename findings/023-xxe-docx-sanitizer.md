# Finding: XXE in `docx-sanitizer.py` — Native XML Parsing Without defusedxml

## Severity
LOW

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:N/A:L = 4.3 (billion-laughs DoS)

## Affected Code
- **File:** `ee/features/conversions/python/docx-sanitizer.py:146`
- **Import:** `import xml.etree.ElementTree as ET` at line 35

## Description
The DOCX sanitizer script uses Python's native `xml.etree.ElementTree.parse()` (`ET.parse()`) to parse XML files extracted from user-uploaded DOCX documents, without using the `defusedxml` library.

Verification: Python 3's `xml.etree.ElementTree.parse()` does NOT resolve external entities by default. This means classic XXE file read and SSRF attacks are NOT exploitable. However, billion-laughs (recursive entity expansion) DoS is still possible because entity expansion is enabled by default in `xml.etree.ElementTree`.

This is a defense-in-depth finding. The AI SAST explicitly stated "XXE: ✅ Not found" — a false negative.

## Proof of Concept
1. Craft a DOCX file with a billion-laughs payload in `word/document.xml`:
   ```xml
   <?xml version="1.0"?>
   <!DOCTYPE lolz [
     <!ENTITY lol "lol">
     <!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
     <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
     ...
   ]>
   <root>&lolz;</root>
   ```
2. Upload the crafted DOCX through any document upload endpoint
3. The document conversion pipeline calls `docx-sanitizer.py` which parses the XML with `ET.parse()`
4. Entity expansion exhausts server memory during parsing

## Impact
Billion-laughs DoS via crafted DOCX files during document conversion. Requires authenticated upload. The Python process runs on the server and could consume significant memory before the parser terminates.

## Evidence
```python
# ee/features/conversions/python/docx-sanitizer.py:145-147
_register_all_namespaces(path)
tree = ET.parse(path)  # No defusedxml — billion-laughs DoS still possible
root = tree.getroot()
```

```python
# Standard xml.etree.ElementTree (not defusedxml)
import xml.etree.ElementTree as ET  # at module level, line 35
```

## Methodology
- **First found by:** Semgrep custom rule `papermark-python-xxe-xml-parse` (04-custom-rules.md)
- **Confirmed by:** AI SAST explicitly stated "XXE: ✅ Not found" — false negative
- **Confirmed by:** Reachability analysis confirmed DOCX conversion pipeline

## Chain Potential
- **Chain 4 (DOCX XXE → Cloud Credentials):** Originally assessed as SSRF-to-cloud-metadata path, but mitigated because Python 3 does not resolve external entities by default. Chain 4 is LOWERED to DoS-only impact.

## Remediation
Replace `import xml.etree.ElementTree as ET` with `import defusedxml.ElementTree as ET` from the `defusedxml` library. This adds protection against both XXE and billion-laughs attacks:
```python
from defusedxml import ElementTree as ET
```
