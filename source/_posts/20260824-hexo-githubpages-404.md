---
title: "[Debugging] Hexo Page deployed but error 404?"
date: 2026-08-24 02:34:00
categories:
- Debugging
---

# Why it happens?

Some page does not exist (case: this is a first page) due to build error.

Many causes exist. Example is using colon or brackets in title without quoting. This will be the error.

```
ERROR Process failed: _posts/20260824-hexo-page-not-deploying.md
YAMLException: bad indentation of a mapping entry (1:17)
```

If encountering this, please add quotes in the title!