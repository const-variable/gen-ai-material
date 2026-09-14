# gen-ai-material

## part_1 — BBC News Analysis with LangChain + Ollama

so the goal here was to run topic classification + summarization + entity extraction on the entire BBC news dataset (2,225 articles) without dying from API rate limits, so went full local with Ollama instead of hitting a paid API.

what's actually happening in the notebook:
- using LangChain + Ollama to run everything locally, no API keys needed
- two models doing different jobs based on how hard the task is: `llama3.2:1b` (fast/tiny) for classification since it's basically a one-word answer, and `llama3.2:3b` (bigger) for summaries + entities since those need more brains
- loads the BBC dataset (tab separated csv, 2225 rows across business/entertainment/politics/sport/tech)
- built 3 separate LangChain chains first (classify, summarize, extract entities) and sanity-checked each on one article
- then for the full run, merged all 3 tasks into **one combined LLM call per article** instead of 3 separate ones — cuts total calls by ~3x which matters a lot when you're running thousands of articles locally
- added progress logging (prints every 10 rows with speed + ETA) and checkpointing every 100 rows to csv so a multi-hour run doesn't get wrecked if something crashes
- has a fallback: if the combined call returns broken JSON, it just falls back to the 3 individual chains for that one article instead of failing
- at the end it checks how often the model's predicted topic actually matches the original dataset label (was getting ~83% agreement on the test sample)
- exports everything to both csv and a clean JSON file (article id, title, trimmed text, topic, summary, entities)

tldr: local SLMs + LangChain chains + smart batching so you can process a whole dataset without burning API credits or waiting forever.
