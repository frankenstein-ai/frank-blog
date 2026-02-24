+++
date = '2026-02-24T11:19:18-03:00'
draft = false
title = 'Automating Blog Post Generation with a Reusable LLM Workflow: lessons from 11 quick fixes'
+++

## The problem

We wanted to auto-generate blog posts for Find Workers using a reusable GitHub Actions workflow that calls an LLM. We assumed it would be simple: add a workflow that runs on push, point it at the reusable workflow, provide the model and secrets, and let the generator create posts. It wasn’t.

In practice we hit a string of edge cases: the reusable-workflow path was wrong, model naming conventions differed between providers, required secrets were missing, and we needed a sentinel value to “remove” the temperature parameter exposed by the reusable workflow.

Over 11 commits in about 24 hours we went from “it doesn’t run” to “it runs, but uses the wrong model” to “it used OpenAI gpt-5-mini with no temperature.” Below I explain what we changed, why, and practical lessons for anyone wiring LLM-driven content generation into CI.
