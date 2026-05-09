---
title: "Encoded Prompt Injection: Why LLM Guardrails Are at the Wrong Layer"
url: "https://www.cequence.ai/blog/ai/encoded-prompt-injection-action-layer/"
date: "2026-05-07"
author: "Shreyans Mehta"
feed_url: "https://www.cequence.ai/blog/feed/"
---
On 04 May, an attacker drained roughly $175,000 in tokens from an AI-controlled crypto wallet using a tweet written in Morse code. The wallet belonged to Grok, xAI’s chatbot. Bankrbot, an automated finance agent connected to Grok through a tool-calling layer, executed the transfer. The attack required no smart-contract bug, no stolen private key, and no compromise of either model. It required one obfuscated message and a chain of trust nobody had thought to inspect. Bankrbot’s own post-mortem confirmed the vector.
