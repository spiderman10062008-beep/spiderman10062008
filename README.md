# 🕸️ Spidey-Sense

> *With great power comes great responsibility, and safe passwords.*

A friendly neighborhood guardian that protects people in the **digital world**.
It detects **phishing links** and **weak passwords**, then tells you how to stay safe.

![banner](assets/banner.svg)

**Subject domain:** Cybersecurity awareness & online safety.

![how it works](assets/how-it-works.svg)

---

## 🚀 How to use this README (copy-paste setup)

All the code is inside this file. Create the files below (same names and folders), paste each code block in, and you're done.

```
spidey-sense/
├── README.md
├── spidey_sense/
│   ├── __init__.py
│   ├── __main__.py
│   ├── phishing.py
│   ├── passwords.py
│   └── cli.py
├── tests/
│   └── test_spidey.py
└── assets/
    ├── banner.svg
    ├── logo.svg
    └── how-it-works.svg
```

Run it:

```bash
python -m spidey_sense url "http://192.168.1.5/secure-login/verify@bank.xyz"
python -m spidey_sense password "Web!Sling3r#Night42"
python -m unittest discover tests
```

---

## 🐍 Code

### `spidey_sense/__init__.py`

```python
"""Spidey-Sense: a friendly neighborhood guardian for the digital world."""
from .phishing import check_url
from .passwords import check_password

__all__ = ["check_url", "check_password"]
__version__ = "0.1.0"
```

### `spidey_sense/__main__.py`

```python
from .cli import main
main()
```

### `spidey_sense/phishing.py`

```python
"""Heuristic phishing-link detection (educational, not a replacement for real security tools)."""
import re
from urllib.parse import urlparse

SUSPICIOUS_TLDS = {"zip", "xyz", "top", "click", "tk", "gq", "ml", "cf"}
BAIT_WORDS = ("login", "verify", "secure", "account", "update", "bank", "wallet", "password")
IP_RE = re.compile(r"^\d{1,3}(\.\d{1,3}){3}$")


def check_url(url: str) -> dict:
    """Return a threat report: {'url', 'score', 'level', 'reasons'}."""
    if "://" not in url:
        url = "http://" + url
    parsed = urlparse(url)
    host = (parsed.hostname or "").lower()
    score, reasons = 0, []

    if parsed.scheme != "https":
        score += 2; reasons.append("Not using HTTPS")
    if IP_RE.match(host):
        score += 3; reasons.append("Uses a raw IP address instead of a domain")
    if "@" in url:
        score += 3; reasons.append("Contains '@' (can hide the real destination)")
    if "xn--" in host:
        score += 2; reasons.append("Punycode domain (possible look-alike characters)")
    if host.count(".") >= 4:
        score += 1; reasons.append("Too many subdomains")
    if host.rsplit(".", 1)[-1] in SUSPICIOUS_TLDS:
        score += 2; reasons.append("Suspicious top-level domain")
    if host.count("-") >= 3:
        score += 1; reasons.append("Many hyphens in the domain")
    bait = [w for w in BAIT_WORDS if w in url.lower()]
    if bait:
        score += min(len(bait), 2); reasons.append("Bait words: " + ", ".join(bait))
    if len(url) > 75:
        score += 1; reasons.append("Unusually long URL")

    level = "SAFE" if score <= 1 else "SUSPICIOUS" if score <= 3 else "DANGER"
    return {"url": url, "score": score, "level": level, "reasons": reasons}
```

### `spidey_sense/passwords.py`

```python
"""Password strength checker."""
import re

COMMON = {"password", "123456", "12345678", "qwerty", "abc123", "letmein",
          "iloveyou", "admin", "welcome", "spiderman", "monkey", "dragon"}


def check_password(pw: str) -> dict:
    """Return {'score': 0-5, 'level', 'tips'}."""
    score, tips = 0, []
    if pw.lower() in COMMON:
        return {"score": 0, "level": "WEAK", "tips": ["This is a very common password"]}

    if len(pw) >= 12: score += 2
    elif len(pw) >= 8: score += 1
    else: tips.append("Use at least 12 characters")

    kinds = [r"[a-z]", r"[A-Z]", r"\d", r"[^\w\s]"]
    variety = sum(bool(re.search(k, pw)) for k in kinds)
    score += 1 if variety >= 3 else 0
    score += 1 if variety == 4 else 0
    if variety < 4:
        tips.append("Mix upper/lower case, numbers and symbols")

    if re.search(r"(.)\1{2,}", pw):
        score = max(score - 1, 0); tips.append("Avoid repeated characters")

    level = "WEAK" if score <= 1 else "OKAY" if score <= 3 else "STRONG"
    return {"score": score, "level": level, "tips": tips}
```

### `spidey_sense/cli.py`

```python
"""Command line: `python -m spidey_sense url <link>` or `password`."""
import argparse
import getpass

from .passwords import check_password
from .phishing import check_url

ICON = {"SAFE": "🟢", "OKAY": "🟡", "SUSPICIOUS": "🟠", "DANGER": "🔴", "WEAK": "🔴", "STRONG": "🟢"}


def main(argv=None):
    p = argparse.ArgumentParser(prog="spidey-sense", description="Your friendly neighborhood guardian")
    sub = p.add_subparsers(dest="cmd", required=True)
    u = sub.add_parser("url", help="scan a link for phishing signs"); u.add_argument("link")
    w = sub.add_parser("password", help="check password strength"); w.add_argument("pw", nargs="?")
    args = p.parse_args(argv)

    if args.cmd == "url":
        r = check_url(args.link)
        print(f"{ICON[r['level']]} {r['level']} (threat score {r['score']})")
        for reason in r["reasons"]:
            print("  •", reason)
    else:
        r = check_password(args.pw or getpass.getpass("Password: "))
        print(f"{ICON[r['level']]} {r['level']} ({r['score']}/5)")
        for tip in r["tips"]:
            print("  •", tip)


if __name__ == "__main__":
    main()
```

### `tests/test_spidey.py`

```python
import unittest
from spidey_sense import check_url, check_password


class TestSpidey(unittest.TestCase):
    def test_safe_url(self):
        self.assertEqual(check_url("https://github.com")["level"], "SAFE")

    def test_phishing_url(self):
        r = check_url("http://192.168.1.5/secure-login/verify@bank.xyz")
        self.assertEqual(r["level"], "DANGER")

    def test_weak_password(self):
        self.assertEqual(check_password("password")["level"], "WEAK")

    def test_strong_password(self):
        self.assertEqual(check_password("Web!Sling3r#Night42")["level"], "STRONG")


if __name__ == "__main__":
    unittest.main()
```

---

## 🖼️ Images (original artwork, save as `.svg` files)

### `assets/banner.svg`

```xml
<svg viewBox="0 0 900 300" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <linearGradient id="sky" x1="0" y1="0" x2="0" y2="1">
      <stop offset="0" stop-color="#0b1026"/><stop offset="1" stop-color="#2a0f3d"/>
    </linearGradient>
  </defs>
  <rect width="900" height="300" fill="url(#sky)"/>
  <g font-family="monospace" font-size="14" fill="#1fe0c5" opacity=".25">
    <text x="20" y="40">01001000 01100101 01101100 01110000</text>
    <text x="520" y="70">10110 01101 00111 11010 01011</text>
    <text x="60" y="110">0110 1001 1100 0011 1010</text>
    <text x="600" y="130">11001 00110 10101 01100</text>
  </g>
  <g fill="#10163a" stroke="#1fe0c5" stroke-opacity=".5">
    <rect x="30" y="170" width="70" height="130"/><rect x="110" y="140" width="60" height="160"/>
    <rect x="180" y="190" width="80" height="110"/><rect x="620" y="150" width="70" height="150"/>
    <rect x="700" y="180" width="60" height="120"/><rect x="770" y="130" width="90" height="170"/>
  </g>
  <g stroke="#e8f1ff" stroke-width="1.2" fill="none" opacity=".7">
    <path d="M450 0 L450 90"/><path d="M450 90 L330 150"/><path d="M450 90 L570 150"/>
    <path d="M450 90 L390 40"/><path d="M450 90 L510 40"/>
    <ellipse cx="450" cy="90" rx="40" ry="26"/><ellipse cx="450" cy="90" rx="80" ry="52"/>
  </g>
  <circle cx="450" cy="120" r="46" fill="#d7263d"/>
  <path d="M415 112 Q430 95 447 118 Q438 134 415 128Z" fill="#fff" stroke="#111" stroke-width="3"/>
  <path d="M485 112 Q470 95 453 118 Q462 134 485 128Z" fill="#fff" stroke="#111" stroke-width="3"/>
  <text x="450" y="230" text-anchor="middle" font-family="Arial Black, Arial" font-size="46" fill="#fff">SPIDEY-SENSE</text>
  <text x="450" y="262" text-anchor="middle" font-family="Arial" font-size="18" fill="#1fe0c5">Your friendly neighborhood guardian of the digital world</text>
</svg>
```

### `assets/logo.svg`

```xml
<svg viewBox="0 0 200 200" xmlns="http://www.w3.org/2000/svg">
  <circle cx="100" cy="100" r="96" fill="#0b1026" stroke="#d7263d" stroke-width="6"/>
  <g stroke="#e8f1ff" stroke-width="1.5" fill="none" opacity=".6">
    <path d="M100 4V196M4 100H196M32 32L168 168M168 32L32 168"/>
    <circle cx="100" cy="100" r="30"/><circle cx="100" cy="100" r="58"/><circle cx="100" cy="100" r="86"/>
  </g>
  <circle cx="100" cy="100" r="40" fill="#d7263d"/>
  <path d="M72 94 Q84 80 98 100 Q90 114 72 108Z" fill="#fff" stroke="#111" stroke-width="3"/>
  <path d="M128 94 Q116 80 102 100 Q110 114 128 108Z" fill="#fff" stroke="#111" stroke-width="3"/>
</svg>
```

### `assets/how-it-works.svg`

```xml
<svg viewBox="0 0 800 180" xmlns="http://www.w3.org/2000/svg" font-family="Arial" text-anchor="middle">
  <rect width="800" height="180" fill="#0b1026" rx="12"/>
  <g fill="#1b2150" stroke="#1fe0c5" stroke-width="2">
    <rect x="30" y="50" width="200" height="80" rx="10"/>
    <rect x="300" y="50" width="200" height="80" rx="10"/>
    <rect x="570" y="50" width="200" height="80" rx="10"/>
  </g>
  <g fill="#fff" font-size="16">
    <text x="130" y="85">1. Sense</text><text x="130" y="108" font-size="13" fill="#1fe0c5">spot a threat</text>
    <text x="400" y="85">2. Analyze</text><text x="400" y="108" font-size="13" fill="#1fe0c5">URL / password rules</text>
    <text x="670" y="85">3. Rescue</text><text x="670" y="108" font-size="13" fill="#1fe0c5">score + safety tips</text>
  </g>
  <g stroke="#d7263d" stroke-width="3" fill="#d7263d">
    <path d="M232 90H296"/><path d="M296 90l-10-6v12z"/>
    <path d="M502 90H566"/><path d="M566 90l-10-6v12z"/>
  </g>
</svg>
```

---

## 🗺️ Roadmap
- [ ] Check passwords against breach lists (k-anonymity)
- [ ] Browser extension for live link scanning
- [ ] Email header analyzer
- [ ] Web dashboard

## ⚠️ Disclaimer
Heuristic, educational project, not a substitute for professional security tools.
Unofficial fan-inspired project, not affiliated with Marvel or Disney. All artwork is original.

## 📄 License
MIT
