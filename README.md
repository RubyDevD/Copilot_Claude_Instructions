# Claude / Copilot Global Instructions for Rails

This repository contains a shared instruction file for AI tools like **Claude** and **GitHub Copilot** to enforce consistent, secure, and production-grade Ruby on Rails development practices.

---

## 📌 Purpose

The goal of this file is to ensure that any AI-generated code:

* Follows Rails best practices
* Is secure by default (no shortcuts)
* Is production-ready from day one
* Avoids common vulnerabilities (SQL injection, XSS, CSRF, etc.)
* Maintains consistency across the team

This is **not optional guidance** — it is a strict rule set.

---

## 📂 File Overview

* `CLAUDE.md` → Global rules and standards for Rails development
  (Place it in `~/.claude/CLAUDE.md`)

---

## ⚙️ Setup Instructions

### 1. Copy the file to your local machine

```bash
mkdir -p ~/.claude
cp CLAUDE.md ~/.claude/CLAUDE.md
```

---

### 2. Restart your AI tool

Restart:

* Claude Desktop (if using)
* VS Code (if using Copilot / extensions)

---

### 3. Start using normally

From now on, all AI-generated code will follow these rules automatically.

---

## 🧠 What This Enforces

The rules cover:

* Rails architecture (MVC, fat models, skinny controllers)
* Security best practices (authentication, tokens, sessions)
* Database integrity and migrations
* API standards and rate limiting
* Background jobs and idempotency
* File uploads and validations
* Logging, monitoring, and secrets handling
* CI/CD and deployment safety

See full rules here: 

---

## ⚠️ Important Notes

* These rules are **strict and non-negotiable**
* AI should not bypass them for convenience
* If something cannot be done securely, it must be explicitly flagged
* Code may include `TODO [SECURITY-*]` or `TODO [PERF]` markers
* Expect slightly longer and more detailed AI responses

---

## 👥 Team Usage & Contribution Guidelines

* All team members should use this file locally for consistent AI-generated code
* The file is **version-controlled**, so updates are shared across the team

---

### 🔄 Updating the Rules

Team members are encouraged to:

* Improve existing rules
* Add new best practices
* Fix gaps or inconsistencies

---

### 🛠️ How to Update

1. Make changes to `CLAUDE.md`
2. Commit your changes:

   ```bash
   git add CLAUDE.md
   git commit -m "Improve AI rules: <short description>"
   ```
3. Push to repository:

   ```bash
   git push origin <branch-name>
   ```
4. Create a Pull Request (recommended for review)

---

### ⚠️ Rules for Updates

* Do NOT weaken security rules without strong justification
* Always explain **why** the change is needed
* Prefer **adding improvements** over removing rules
* Ensure changes do not break existing workflows

---

### 🔁 Syncing Changes

After pulling latest changes, update your local configuration:

```bash
cp CLAUDE.md ~/.claude/CLAUDE.md
```

Then restart your AI tools (Claude / VS Code) to apply updates.

---

## 🔍 Recommended Verification Tools

Run these regularly to maintain code quality and security:

```bash
bundle audit check --update   # Check for vulnerable gems
brakeman -A                   # Static security analysis
```

---

## 🚀 Why This Matters

Without guardrails, AI tools can:

* Introduce security vulnerabilities
* Generate inconsistent or non-idiomatic code
* Ignore production constraints

This setup ensures:

* Safer code
* Faster code reviews
* Fewer production issues
* Better team alignment

---

## 📬 Contributions

If you want to improve these rules:

* Create a Pull Request
* Clearly explain the reason for the change
* Focus on security, performance, or maintainability

---

## ✅ Summary

This setup turns AI into a **senior Rails engineer with strict standards**, not just a code generator.

Use it consistently — and it will prevent costly mistakes in production.
