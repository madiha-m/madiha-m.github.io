---
layout: post
title: "🐍 Fixing 'SyntaxError: Non-UTF-8 code starting with '\\xff'' in Python"
date: 2025-07-12 14:00:00 +0500
categories: [Python, BugFix]
tags: [python, utf-8, encoding, error, windows]
---

👋 **Hi again!**  
Today I ran into a frustrating error while running a Python file. It looked scary at first — but turned out to be a simple encoding issue. Here's what happened, what I tried, and how I finally fixed it. 🧩

---

## 💥 The Problem

I ran this Python script using:

```bash
python my_script.py
```

And got hit with this error:

```bash
SyntaxError: Non-UTF-8 code starting with '\xff' in file my_script.py on line 1
```

---

## 🧪 What I Tried (That Didn’t Help)

At first, I thought I had a typo or some invalid character in my code.

So I tried:

- Rewriting the first few lines
- Copy-pasting the code into a new .py file
- Checking for BOM (byte order mark) issues manually

Still got the same error 😤

---

## 🔍 What Was Actually Wrong

The issue was not in the code itself, but in the file encoding.

Python expects source files to be saved in UTF-8 by default. But my file was saved in a different encoding — probably ANSI or Windows-1252.

This usually happens when:

- The file is edited with Notepad or some older editor
- You download or copy code from external sources (email attachments, ZIPs, etc.)

---

## ✅ What Finally Worked

I opened the file in VS Code, and changed the encoding:

📌 Steps:

1. Open the file in VS Code
2. Click the UTF-8 label in the bottom-right corner
3. Click "Reopen with Encoding" → select Windows 1252 or ISO 8859-1 (whichever works)
4. Then click "Save with Encoding" → select UTF-8
5. Save and re-run the script!

Boom 💥 — no more syntax error.

---

## ⏳ What’s Next?

If you're working with multiple files or a team, make sure everyone saves files in UTF-8 to avoid this hidden bug. Add this line at the top of your file to be explicit:

```bash
# -*- coding: utf-8 -*-
```

Or configure your editor to always use UTF-8.

---

## 🙋‍♀️ Final Note

This issue happened on:

- 💻 Windows 11
- 🐍 Python 3.10
- 📄 File saved from an external source

Let me know if this helped — or if you’ve had a similar experience!
📝 Read this blog on [madiha-m.github.io](https://madiha-m.github.io)

— Madiha 💙
