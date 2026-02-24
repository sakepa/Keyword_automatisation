# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Educational repository from **Data Marketing Labs** (Vincent Terrasi) for automating keyword research and simulating Google's Search Generative Experience (SGE). The repo contains Google Colab notebook scripts (exported as `.py`) and French-language course materials on SEO optimization with AI.

## Repository Structure

- **Python scripts** (Colab notebooks): Designed to run in Google Colab, not as standalone scripts. They use `!pip install` magic commands and `google.colab` auth.
  - `data_marketing_labs_large_language_model_search_optimization_sge_openai_and_palm2_for_students (1).py` — Main SGE simulator. Combines Google Search API + OpenAI (GPT-3.5) + Gemini (Vertex AI) to generate queries, extract semantic entities, and produce keyword reports as Excel files.
  - `data_marketing_labs_gemini_text_generation_sge_for_students.py` — Content generation and quality refinement using Gemini. Generates articles then iteratively checks accuracy, removes fluff, and improves structure.
- **Markdown files** (`.md`): French-language course notes covering SGE methodology, E-E-A-T optimization, keyword extraction, content workflows, and advanced PaLM 2 strategies.
- `searchqualityevaluatorguidelines.pdf` — Google's Search Quality Evaluator Guidelines reference document.

## SGE Simulator Architecture (Main Script)

The core workflow in the large script follows this pipeline:

1. **Query Generation** — Uses Gemini (Vertex AI `gemini-1.5-pro-001`) via `generate_related_queries()` to expand a seed topic into multiple search queries
2. **Parallel LLM Querying** — `query_bard()` calls both Gemini and OpenAI in parallel using `concurrent.futures` threading to generate multiple responses per query
3. **Entity Extraction** — `extract_entities()` uses OpenAI (`gpt-3.5-turbo`) to pull domain-specific named entities from each response
4. **Google Search Fusion** — `extract_main_content()` fetches top Google results via Custom Search API, extracts page content with `trafilatura`, and summarizes with `sentence_transformers` + KMeans clustering
5. **Entity Unification** — `unify_similar_entity()` uses `fuzzywuzzy` to deduplicate similar entities across all responses
6. **Analysis & Reporting** — `analyze_entity()` aggregates entity frequencies; `generate_report()` produces a detailed topic report; results export to Excel

## Required API Keys and Configuration

The scripts require these credentials (set as Colab secrets or inline):
- `GOOGLE_SEARCH_API` — Google Custom Search API key
- `GOOGLE_CSE_ID` — Google Custom Search Engine ID
- `OPENAI_API` — OpenAI API key
- `PROJECT` — Google Cloud project ID (for Vertex AI)
- `LOCATION` — Google Cloud region (e.g., `us-central1`)

## Dependencies

```
google-cloud-aiplatform
openai
fuzzywuzzy
retrying
trafilatura
summarizer
sentence_transformers
scikit-learn
pandas
tenacity
google-api-python-client
```

## Running the Scripts

These are Colab notebooks. To run them:
1. Open in Google Colab (links in script headers)
2. Run the setup cell first (`!pip install` commands) — Colab may require a runtime restart after installing `google-cloud-aiplatform`
3. Configure API keys and project settings
4. Execute cells sequentially

## Key Parameters

In the SGE simulator, two parameters control generation scope:
- `number_of_queries` — Number of query variations to generate from the seed topic
- `number_of_generations` — Number of LLM responses per query (higher = better coverage but more API cost)

## Language

Course materials and code comments are primarily in **French**. Function names and API interactions are in English.
