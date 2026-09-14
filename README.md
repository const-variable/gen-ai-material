# gen-ai-material

## why local Ollama SLMs instead of a hosted API

Both parts of this assignment run entirely on **local small language models via Ollama** instead of a hosted API like Groq. Groq's free tier has fairly tight per-minute/per-day token and request limits, and this assignment means running an LLM over every row of a dataset (2,225 news articles in part_1, and job postings in part2) — a run of that size blows through those limits fast and the job just dies partway through. Going local with Ollama sidesteps that entirely: no rate limits, no API keys, no cost, and the run can go as long as it needs to.

The tradeoff is raw speed (CPU-only local inference is slower per-call than a hosted API), which is why both notebooks lean on a few tricks to make full-dataset runs practical:
- **two-tier models** — a tiny/fast model (`llama3.2:1b`) for cheap one-word-ish outputs (classification), a bigger model (`llama3.2:3b`) for tasks that need more reasoning (summarization, entity/requirement extraction)
- **combined calls** — instead of separate LLM calls per task, all tasks for a given row are merged into **one call per record**, cutting total calls ~3x
- **checkpointing** — results are saved to CSV periodically (every 100 rows) so a multi-hour run survives a crash instead of losing everything
- **capped input length** — long text gets clipped before being sent to the model to keep generation fast
- **fallback parsing** — if the combined call returns malformed JSON, the code falls back to the individual per-task chains for just that row instead of failing the whole run

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
