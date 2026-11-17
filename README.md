# keunchang.log

https://rmsckd1640.github.io

Jekyll로 만든 개인 블로그. `main`에 푸시하면 GitHub Actions가 빌드해 GitHub Pages로 배포한다.

## 글 쓰기

`_posts/YYYY-MM-DD-제목.md`

```yaml
---
title: "글 제목"
date: 2026-11-17 21:00:00 +0900
series: "알고리즘"   # 시리즈에 속할 때만
tags: ["Java"]
summary: "목록에 보일 한두 줄 요약"
---
```

이미지는 `assets/img/posts/<글 슬러그>/`에 둔다.

## 로컬 미리보기

```bash
bundle install
bundle exec jekyll serve
```
