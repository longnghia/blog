---
date: "{{ .Date }}"
draft: true
title: "{{ replace .File.ContentBaseName `-` `` | title }}"
summary: "{{ replace .File.ContentBaseName `-` `` | title }}"
categories:
  - Code
tags:
  - ssh
---
