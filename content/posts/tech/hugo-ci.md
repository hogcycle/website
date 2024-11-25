---
date: '2024-11-25T14:51:16-05:00'
draft: true
title: 'Hugo CI'
---

git submodule init
git submodule update

 rsync -az --delete ~/website/public/* /var/www/gysling/
