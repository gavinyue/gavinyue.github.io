---
title: "TIL: run hugo server against another directory"
date: 2026-07-25
tags: [hugo]
---

`hugo server --source /path/to/site` runs the dev server without cd-ing into the project. Handy when your terminal lives in one repo and your blog in another:

```sh
hugo server --source ~/workspace/gavinyue-web --port 1414
```

Config changes hot-reload too, including theme switches.
