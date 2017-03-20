---
layout: post
title: mkdir 循环创建目录树
date: 2014-05-29
---

```bash
for dir in $(seq -w 1 111)
do
  mkdir -p 2007${dir}.{33,44}/{aa,bb,cc}
done
```