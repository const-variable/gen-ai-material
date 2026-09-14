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

## part2 — Job Postings Analysis with LangChain + Ollama

goal here was to take raw job postings and pull out structured info: which domain the role belongs to, what skills/tools it needs, and the education + experience requirements — again all local via Ollama, same reasoning as part_1 about not burning API credits.

what's actually happening in the notebook:
- same two-model split as part_1: `llama3.2:1b` for the quick category label, `llama3.2:3b` for the heavier requirements extraction (skills/education/experience as structured JSON)
- loads `job_title_des.csv` (2,277 job title + description rows), tidies columns, assigns a `Job_ID`
- built the classify chain and the extract chain separately first, sanity-checked both on one posting
- job descriptions get clipped to ~4000 chars before hitting the model so long postings don't tank speed
- for the actual run, classification + skills + education + experience all get pulled in **one combined call per posting** (same "merge calls" trick as part_1), with a fallback to the two separate chains if the combined JSON doesn't parse
- a `parse_requirements` helper handles messy/partial JSON from the small model so a bad response doesn't kill the row
- progress logging every 10 rows + CSV checkpointing every 100 rows, same as part_1
- `NUM_JOBS` controls how much of the dataset to run (set to 25 for this run; `None` processes all 2,277)
- exports results to both `job_analysis_full.csv` and `job_analysis_output.json` (title, trimmed description, predicted category, skills, education, experience)

tldr: same local SLM + LangChain + combined-call pattern as part_1, applied to job postings instead of news articles — pulls structured hiring requirements out of messy free-text listings.
