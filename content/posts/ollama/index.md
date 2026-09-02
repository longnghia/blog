---
date: "2026-08-05T12:18:50+07:00"
draft: true
title: "Exposing Ollama on the Local Network"
summary: "Binding Ollama to all interfaces and hitting the generate API from another machine."
categories:
  - Code
tags:
  - ollama
  - llm
  - selfhosted
---

## open port on all interfaces to make it accessible on local network (not only localhost)

## commands

```sh
curl http://172.16.0.35:11434/api/generate -d '{
     "model": "qwen3-coder:30b",
     "prompt": "Why is the sky blue?"
   }'
```
