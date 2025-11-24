# Automated Keyword Research Workflow - Complete Technical Documentation

## Executive Summary

This document provides comprehensive technical documentation for an Automated Keyword Research Workflow designed to enable **100% reproduction** with any modern Large Language Model (LLM) technology. The original implementation uses Google Vertex AI Palm2 (chat-bison models), but this documentation includes complete replacement guidelines for OpenAI, Anthropic Claude, Mistral, and local models.

---

## Table of Contents

1. [Workflow Overview](#1-workflow-overview)
2. [Technical Stack](#2-technical-stack)
3. [Configuration Variables](#3-configuration-variables)
4. [Complete Function Reference](#4-complete-function-reference)
5. [Data Flow Specifications](#5-data-flow-specifications)
6. [State Machine Definition](#6-state-machine-definition)
7. [LLM Replacement Guidelines](#7-llm-replacement-guidelines)
8. [Error Handling Patterns](#8-error-handling-patterns)
9. [Prompt Templates](#9-prompt-templates)
10. [Dependencies and Installation](#10-dependencies-and-installation)

---

## 1. Workflow Overview

### 1.1 High-Level Process Flow

```
[SEED TOPIC] --> [Query Generation] --> [Google Search] --> [Content Extraction]
                                                                    |
                                                                    v
[Topical Map Report] <-- [Entity Analysis] <-- [Entity Extraction] <-- [LLM Summarization]
```

### 1.2 Seven-Step Pipeline

| Step | Name | Description | Input | Output |
|------|------|-------------|-------|--------|
| 1 | Seed Topic Input | User provides initial topic/keyword | String | `topic` variable |
| 2 | Query Generation | LLM expands seed into related queries | `topic` | List of queries |
| 3 | Google Search | Search API retrieves top results | Query list | Search results (title, link, snippet) |
| 4 | Content Extraction | Trafilatura extracts page content | URLs | Raw text content |
| 5 | LLM Summarization | Dual summarization (extractive + abstractive) | Raw text | Condensed summaries |
| 6 | Entity Extraction | LLM identifies SEO entities | Summaries | Entity list with prominence |
| 7 | Report Generation | Creates topical map and gap analysis | Entity DataFrame | Final report |

---

## 2. Technical Stack

### 2.1 Core Dependencies

```python
# LLM Integration (Original - Google Vertex AI)
import vertexai
from vertexai.preview.language_models import ChatModel

# Web Search
from googleapiclient.discovery import build  # Google Custom Search API

# Web Scraping
import trafilatura

# Text Processing
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.metrics.pairwise import cosine_similarity
from sklearn.cluster import KMeans
from sentence_transformers import SentenceTransformer
import numpy as np

# Entity Deduplication
from fuzzywuzzy import fuzz, process

# Retry Logic
from tenacity import retry, wait_random_exponential, stop_after_attempt

# Concurrency
from concurrent.futures import ThreadPoolExecutor

# Data Handling
import pandas as pd
import ast
```

### 2.2 LLM Model Specifications

#### Standard Model (chat-bison)
```python
model_name = "chat-bison"
parameters = {
    "temperature": 0.5,
    "max_output_tokens": 600,
    "top_p": 0.8,
    "top_k": 40
}
```

#### Extended Context Model (chat-bison-32k)
```python
model_name = "chat-bison-32k"
parameters = {
    "temperature": 0.4,
    "max_output_tokens": 8000,
    "top_p": 0.8,
    "top_k": 40
}
```

### 2.3 Embedding Model

```python
model = SentenceTransformer('paraphrase-MiniLM-L6-v2')
# Output: 384-dimensional dense vectors
```

---

## 3. Configuration Variables

```python
# Google Cloud Configuration
PROJECT = "your-gcp-project-id"
LOCATION = "us-central1"  # or your preferred region

# Google Custom Search API
GOOGLE_API_KEY = "your-google-api-key"
CUSTOM_SEARCH_ENGINE_ID = "your-cse-id"

# Workflow Parameters
limit = 10           # Number of search results per query
iso = "en"           # Language/region code (ISO 639-1)
topic = "your-seed-keyword"

# Global Counters
bard_call_counter = 0
```

---

## 4. Complete Function Reference

### 4.1 LLM Interface Functions

#### `bard_call(system_prompt, user_prompt)` - Standard LLM Call
```python
@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def bard_call(system_prompt, user_prompt):
    """
    Makes a standard LLM call with retry logic.

    Args:
        system_prompt (str): System context/instructions
        user_prompt (str): User query/request

    Returns:
        str: LLM response text
    """
    global bard_call_counter
    bard_call_counter += 1

    vertexai.init(project=PROJECT, location=LOCATION)
    chat_model = ChatModel.from_pretrained("chat-bison")

    parameters = {
        "temperature": 0.5,
        "max_output_tokens": 600,
        "top_p": 0.8,
        "top_k": 40
    }

    chat = chat_model.start_chat(context=system_prompt)
    response = chat.send_message(user_prompt, **parameters)

    return response.text
```

#### `bard_call_long(system_prompt, user_prompt)` - Extended Context LLM Call
```python
@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def bard_call_long(system_prompt, user_prompt):
    """
    Makes an extended context LLM call for longer outputs (reports, analysis).

    Args:
        system_prompt (str): System context/instructions
        user_prompt (str): User query/request

    Returns:
        str: LLM response text (up to 8000 tokens)
    """
    global bard_call_counter
    bard_call_counter += 1

    vertexai.init(project=PROJECT, location=LOCATION)
    chat_model = ChatModel.from_pretrained("chat-bison-32k")

    parameters = {
        "temperature": 0.4,
        "max_output_tokens": 8000,
        "top_p": 0.8,
        "top_k": 40
    }

    chat = chat_model.start_chat(context=system_prompt)
    response = chat.send_message(user_prompt, **parameters)

    return response.text
```

### 4.2 Search and Content Functions

#### `google_search(query, num_results, lang)` - Google Custom Search
```python
def google_search(query, num_results, lang):
    """
    Performs Google Custom Search and returns formatted results.

    Args:
        query (str): Search query
        num_results (int): Number of results to retrieve (max 10 per call)
        lang (str): Language/region code (ISO 639-1)

    Returns:
        list: List of [title, link, snippet] for each result
    """
    service = build("customsearch", "v1", developerKey=GOOGLE_API_KEY)
    results = (
        service.cse()
        .list(q=query, cx=CUSTOM_SEARCH_ENGINE_ID, num=num_results, gl=lang)
        .execute()
        .get("items", [])
    )

    formatted_results = [
        [result["title"], result["link"], result["snippet"]]
        for result in results
    ]

    return formatted_results
```

#### `get_webpage_text(url)` - Web Content Extraction
```python
def get_webpage_text(url):
    """
    Extracts main text content from a webpage using trafilatura.

    Args:
        url (str): URL of the webpage

    Returns:
        str: Extracted text content (or None if extraction fails)
    """
    downloaded = trafilatura.fetch_url(url)
    text = trafilatura.extract(downloaded)
    return text
```

### 4.3 Summarization Functions

#### `extractive_summary(text, num_sentences)` - TF-IDF Based Summarization
```python
def extractive_summary(text, num_sentences=3):
    """
    Summarizes text by selecting the most relevant sentences using
    CountVectorizer and cosine similarity.

    Algorithm:
    1. Split text into sentences
    2. Vectorize sentences using CountVectorizer
    3. Calculate cosine similarity matrix
    4. Sum similarity scores for each sentence
    5. Select top N sentences by score
    6. Return sentences in original order

    Args:
        text (str): Input text to summarize
        num_sentences (int): Number of sentences in summary (default: 3)

    Returns:
        str: Summarized text
    """
    # Split the text into sentences
    sentences = [sentence.strip() for sentence in text.split('.') if sentence]

    # If there are fewer sentences than desired, return whole text
    if len(sentences) <= num_sentences:
        return ' '.join(sentences)

    # Use CountVectorizer to convert sentences into vectors
    vectorizer = CountVectorizer().fit_transform(sentences)
    vectors = vectorizer.toarray()

    # Calculate cosine similarity
    cosine_matrix = cosine_similarity(vectors)
    scores = cosine_matrix.sum(axis=1)

    # Get indices of sentences with the highest scores
    ranked_indices = scores.argsort()[-num_sentences:][::-1]

    # Sort indices to maintain original order
    ranked_indices = sorted(ranked_indices)

    # Select top sentences
    selected_sentences = [sentences[i] for i in ranked_indices]

    return '. '.join(selected_sentences)
```

#### `abstractive_summary(text, num_clusters)` - Clustering-Based Summarization
```python
def abstractive_summary(text, num_clusters=3):
    """
    Summarizes text using sentence embeddings and KMeans clustering.
    Selects representative sentences from each cluster.

    Algorithm:
    1. Split text into sentences
    2. Generate embeddings using SentenceTransformer
    3. Cluster embeddings using KMeans
    4. Find sentence closest to each cluster center
    5. Return representative sentences

    Args:
        text (str): Input text to summarize
        num_clusters (int): Number of clusters/summary sentences (default: 3)

    Returns:
        str: Summarized text
    """
    # Split into sentences
    sentences = [sentence.strip() for sentence in text.split('.') if sentence]

    # If fewer sentences than clusters, return whole text
    if len(sentences) <= num_clusters:
        return text

    # Generate embeddings
    model = SentenceTransformer('paraphrase-MiniLM-L6-v2')
    embeddings = model.encode(sentences)

    # Cluster sentences
    kmeans = KMeans(n_clusters=num_clusters, random_state=42)
    kmeans.fit(embeddings)
    cluster_centers = kmeans.cluster_centers_

    # Find sentence closest to each cluster center
    summary_sentences = []
    for center in cluster_centers:
        distances = np.linalg.norm(embeddings - center, axis=1)
        closest_idx = np.argmin(distances)
        summary_sentences.append(sentences[closest_idx])

    return '. '.join(summary_sentences)
```

### 4.4 Entity Extraction Functions

#### `extract_entities_bard(response)` - LLM-Based Entity Extraction
```python
def extract_entities_bard(response):
    """
    Uses LLM to extract SEO-relevant entities from text.

    Args:
        response (str): Text to extract entities from

    Returns:
        list: List of extracted entity strings
    """
    system_prompt = """You are a helpful assistant. Your task is to extract
    specific SEO elements such as named entities and domain-specific
    expressions from a text."""

    user_prompt = f"""Can you extract any specific SEO elements from the
    following text: "{response}"? Please respond only with a list of items
    in the following format, removing stop words and the result is a
    Python list []."""

    entities_list = []
    try:
        entities_str = bard_call(system_prompt, user_prompt)
        entities_list = ast.literal_eval(entities_str)
    except SyntaxError:
        entities_list = []

    return entities_list
```

### 4.5 Prominence Calculation

#### Prominence Formula
```python
# Prominence calculation formula:
# Higher prominence for entities appearing earlier in text
prominence = 1 / (response.lower().split().index(entity.lower()) + 1
                  if entity.lower() in response.lower().split()
                  else len(response))

# Example:
# - Entity at position 0: prominence = 1/(0+1) = 1.0
# - Entity at position 4: prominence = 1/(4+1) = 0.2
# - Entity not found: prominence = 1/len(response) (very small)
```

### 4.6 Main Processing Function

#### `query_bard(query, n)` - Main Query Processing Pipeline
```python
def query_bard(query, n):
    """
    Main processing function that orchestrates the entire pipeline for a single query.

    Process:
    1. Validate query
    2. Extract main content from Google search results
    3. Summarize content
    4. Extract entities using LLM
    5. Calculate prominence scores
    6. Build results DataFrame

    Args:
        query (str): Search query to process
        n (int): Query number (for logging)

    Returns:
        pd.DataFrame: DataFrame with columns:
            - Query: Original query
            - Response: Summarized content
            - Entity: Extracted entity
            - Mentions: Count of entity mentions
            - Prominence: Prominence score
    """
    print(f"Querying GPT for query: '{query}'")
    df = pd.DataFrame(columns=['Query', 'Response', 'Entity', 'Mentions', 'Prominence'])

    if not query:  # Check if query is empty
        print("Skipping empty query")
        return pd.DataFrame()  # Return empty DataFrame for empty query

    # Extract content from search results
    data = extract_main_content(query, limit, iso)
    print(data)

    # Build tasks for processing
    tasks = []
    content_all = ""
    for entry in data:
        content_all += entry["snippet"]

    # Summarize aggregated content
    response = extractive_summary(content_all)

    # Extract entities from summarized content
    entities_list = extract_entities_bard(response)

    # Calculate prominence for each entity
    for entity in entities_list:
        if entity:
            mention_count = response.lower().count(entity.lower())
            if mention_count > 0:
                # Calculate prominence based on first position
                prominence = 1 / (response.lower().split().index(entity.lower()) + 1
                                  if entity.lower() in response.lower().split()
                                  else len(response))

                temp_df = pd.DataFrame([{
                    'Query': query,
                    'Response': response,
                    'Entity': entity,
                    'Mentions': mention_count,
                    'Prominence': prominence
                }], index=[0])

                df = pd.concat([df, temp_df], ignore_index=True)

    return df
```

### 4.7 Entity Analysis Functions

#### `analyze_entity(df)` - Entity Aggregation
```python
def analyze_entity(df):
    """
    Aggregates entity metrics across all queries.

    Args:
        df (pd.DataFrame): DataFrame with entity data from query_bard

    Returns:
        pd.DataFrame: Aggregated DataFrame with columns:
            - Entity: Unique entity name
            - Total Mentions: Sum of all mentions
            - Average Prominence: Mean prominence score
    """
    print("Analyzing entity...")

    # Get unique entities (excluding N/A)
    unique_entities = df[df['Entity'] != 'N/A']['Entity'].unique()

    # Initialize results DataFrame
    analysis_df = pd.DataFrame(columns=['Entity', 'Total Mentions', 'Average Prominence'])

    for entity in unique_entities:
        entity_df = df[df['Entity'] == entity]

        # Calculate aggregates
        total_mentions = entity_df['Mentions'].sum()
        avg_prominence = entity_df['Prominence'].mean()

        # Gather contexts
        contexts = entity_df['Response'].unique()
        context_summaries = []

        # Append to results
        temp_df = pd.DataFrame({
            'Entity': [entity],
            'Total Mentions': [total_mentions],
            'Average Prominence': [avg_prominence]
        })
        analysis_df = pd.concat([analysis_df, temp_df], ignore_index=True)

    return analysis_df
```

#### `unify_similar_entity(df, threshold)` - Fuzzy Entity Deduplication
```python
def unify_similar_entity(df, threshold=90):
    """
    Deduplicates similar entities using fuzzy string matching.

    Algorithm:
    1. Get all unique entities
    2. For each entity, find similar entities using fuzz.token_sort_ratio
    3. Replace similar entities with the first match

    Args:
        df (pd.DataFrame): DataFrame containing 'Entity' column
        threshold (int): Similarity threshold (0-100, default: 90)

    Returns:
        pd.DataFrame: DataFrame with unified entities
    """
    df['Entity'] = df['Entity'].astype(str)
    entities = df['Entity'].unique()

    for entity in entities:
        # Find all similar entities
        matches = process.extract(
            entity,
            entities,
            limit=len(entities),
            scorer=fuzz.token_sort_ratio
        )

        # Filter matches above threshold
        similar = [
            match[0] for match in matches
            if match[1] >= threshold and match[0] != entity
        ]

        # Replace similar entities with the main entity
        if similar:
            for similar_entity in similar:
                df['Entity'] = df['Entity'].replace(similar_entity, entity)

    return df
```

### 4.8 Report Generation

#### `generate_report(analysis_df, specific_topic)` - Topical Map Generation
```python
def generate_report(analysis_df, specific_topic):
    """
    Generates a comprehensive topical map report using extended LLM context.

    Args:
        analysis_df (pd.DataFrame): Aggregated entity analysis DataFrame
        specific_topic (str): The main topic/keyword being researched

    Returns:
        str: Generated topical map report
    """
    # Filter entities related to specific topic
    specific_topic_df = analysis_df[
        analysis_df['Entity'].str.contains(specific_topic, case=False, na=False)
    ]

    report = ""
    keywords = ''

    if not specific_topic_df.empty:
        # Build keyword string
        for i, row in specific_topic_df.iterrows():
            keywords += row['Entity'] + ','

        system_prompt_bard = "You are a helpful SEO and content strategy assistant."

        user_prompt = f"""
        Please create a topical map for {specific_topic}, based on the keywords: {keywords}.
        Respect the iso code {iso}
        Visualize the topical gap in table format.

        Step 1:
        Your first task is to function as a topical map architect.
        Your objective is to compile a comprehensive list of nouns, predicates,
        and named entities that semantically relate to a given topic.
        Your map should include at least 50 nouns/predicates associated with [Your Topic].
        These should be categorized into five primary sections.

        Step 2:
        Create a content strategy with pillar pages and supporting articles.

        Step 3:
        Identify content gaps and opportunities.

        Step 4:
        Provide keyword clustering recommendations.

        Step 5:
        Generate a prioritized action plan.
        """

        chat_response = bard_call_long(system_prompt_bard, user_prompt)
        report = chat_response
        print(report)
    else:
        print(f"No entities found containing '{specific_topic}'")

    return report
```

### 4.9 Query Expansion

#### `generate_related_queries(topic)` - LLM Query Expansion
```python
def generate_related_queries(topic):
    """
    Generates related search queries from a seed topic using LLM.

    Args:
        topic (str): Seed topic/keyword

    Returns:
        list: List of related query strings
    """
    system_prompt = """You are an SEO expert. Generate related search queries
    that people might use when researching the given topic."""

    user_prompt = f"""Generate 10 related search queries for the topic: "{topic}".
    Return only a Python list of strings, no explanations."""

    try:
        response = bard_call(system_prompt, user_prompt)
        queries = ast.literal_eval(response)
        return queries
    except:
        return [topic]  # Fallback to original topic
```

---

## 5. Data Flow Specifications

### 5.1 DataFrame Schemas

#### Query Results DataFrame (from `query_bard`)
```python
{
    'Query': str,           # Original search query
    'Response': str,        # Summarized content
    'Entity': str,          # Extracted entity
    'Mentions': int,        # Count of mentions
    'Prominence': float     # Position-based score (0.0 - 1.0)
}
```

#### Analysis DataFrame (from `analyze_entity`)
```python
{
    'Entity': str,              # Unique entity name
    'Total Mentions': int,      # Aggregated mention count
    'Average Prominence': float # Mean prominence score
}
```

### 5.2 Data Pipeline

```
Input: topic (str)
    |
    v
generate_related_queries(topic) --> List[str] queries
    |
    v
For each query:
    google_search(query, limit, iso) --> List[[title, link, snippet]]
        |
        v
    get_webpage_text(link) --> str content (optional)
        |
        v
    extractive_summary(content) --> str summary
        |
        v
    extract_entities_bard(summary) --> List[str] entities
        |
        v
    Calculate prominence for each entity
        |
        v
    query_bard(query, n) --> DataFrame
    |
    v
Concatenate all DataFrames --> Master DataFrame
    |
    v
unify_similar_entity(df, threshold=90) --> Deduplicated DataFrame
    |
    v
analyze_entity(df) --> Analysis DataFrame
    |
    v
generate_report(analysis_df, topic) --> str Report
```

---

## 6. State Machine Definition

### 6.1 Workflow States

```
[IDLE]
    --> [INITIALIZING]
    --> [QUERY_GENERATION]
    --> [SEARCHING]
    --> [EXTRACTING]
    --> [SUMMARIZING]
    --> [ENTITY_EXTRACTION]
    --> [ANALYZING]
    --> [REPORT_GENERATION]
    --> [COMPLETE]
    --> [ERROR] (from any state)
```

### 6.2 State Transitions

| From State | To State | Trigger | Guard Condition |
|------------|----------|---------|-----------------|
| IDLE | INITIALIZING | start() | topic is valid |
| INITIALIZING | QUERY_GENERATION | init_complete() | credentials valid |
| QUERY_GENERATION | SEARCHING | queries_ready() | len(queries) > 0 |
| SEARCHING | EXTRACTING | search_complete() | results exist |
| EXTRACTING | SUMMARIZING | content_ready() | text extracted |
| SUMMARIZING | ENTITY_EXTRACTION | summaries_ready() | summaries valid |
| ENTITY_EXTRACTION | ANALYZING | entities_ready() | entities found |
| ANALYZING | REPORT_GENERATION | analysis_complete() | analysis_df not empty |
| REPORT_GENERATION | COMPLETE | report_ready() | report generated |
| ANY | ERROR | error_occurred() | exception raised |

---

## 7. LLM Replacement Guidelines

### 7.1 OpenAI GPT-4 / GPT-3.5

```python
import openai
from tenacity import retry, wait_random_exponential, stop_after_attempt

# Configuration
openai.api_key = "your-openai-api-key"

@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def openai_call(system_prompt, user_prompt):
    """Replace bard_call with OpenAI API."""
    response = openai.ChatCompletion.create(
        model="gpt-4",  # or "gpt-3.5-turbo"
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        temperature=0.5,
        max_tokens=600,
        top_p=0.8
    )
    return response.choices[0].message.content

@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def openai_call_long(system_prompt, user_prompt):
    """Replace bard_call_long with OpenAI API (extended context)."""
    response = openai.ChatCompletion.create(
        model="gpt-4-turbo",  # or "gpt-4-32k"
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        temperature=0.4,
        max_tokens=8000,
        top_p=0.8
    )
    return response.choices[0].message.content
```

### 7.2 Anthropic Claude

```python
import anthropic
from tenacity import retry, wait_random_exponential, stop_after_attempt

# Configuration
client = anthropic.Anthropic(api_key="your-anthropic-api-key")

@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def claude_call(system_prompt, user_prompt):
    """Replace bard_call with Anthropic Claude API."""
    message = client.messages.create(
        model="claude-sonnet-4-20250514",  # or claude-3-haiku, claude-3-opus
        max_tokens=600,
        system=system_prompt,
        messages=[
            {"role": "user", "content": user_prompt}
        ],
        temperature=0.5,
        top_p=0.8
    )
    return message.content[0].text

@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def claude_call_long(system_prompt, user_prompt):
    """Replace bard_call_long with Anthropic Claude API (extended context)."""
    message = client.messages.create(
        model="claude-sonnet-4-20250514",  # 200K context window
        max_tokens=8000,
        system=system_prompt,
        messages=[
            {"role": "user", "content": user_prompt}
        ],
        temperature=0.4,
        top_p=0.8
    )
    return message.content[0].text
```

### 7.3 Mistral AI

```python
from mistralai.client import MistralClient
from mistralai.models.chat_completion import ChatMessage
from tenacity import retry, wait_random_exponential, stop_after_attempt

# Configuration
client = MistralClient(api_key="your-mistral-api-key")

@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def mistral_call(system_prompt, user_prompt):
    """Replace bard_call with Mistral API."""
    messages = [
        ChatMessage(role="system", content=system_prompt),
        ChatMessage(role="user", content=user_prompt)
    ]
    response = client.chat(
        model="mistral-large-latest",  # or "mistral-medium", "mistral-small"
        messages=messages,
        temperature=0.5,
        max_tokens=600,
        top_p=0.8
    )
    return response.choices[0].message.content

@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def mistral_call_long(system_prompt, user_prompt):
    """Replace bard_call_long with Mistral API (extended context)."""
    messages = [
        ChatMessage(role="system", content=system_prompt),
        ChatMessage(role="user", content=user_prompt)
    ]
    response = client.chat(
        model="mistral-large-latest",  # 32K context
        messages=messages,
        temperature=0.4,
        max_tokens=8000,
        top_p=0.8
    )
    return response.choices[0].message.content
```

### 7.4 Local Models (Ollama)

```python
import requests
from tenacity import retry, wait_random_exponential, stop_after_attempt

# Configuration
OLLAMA_BASE_URL = "http://localhost:11434"

@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def ollama_call(system_prompt, user_prompt):
    """Replace bard_call with local Ollama model."""
    response = requests.post(
        f"{OLLAMA_BASE_URL}/api/generate",
        json={
            "model": "llama3:8b",  # or "mistral", "codellama", etc.
            "prompt": f"{system_prompt}\n\nUser: {user_prompt}\n\nAssistant:",
            "options": {
                "temperature": 0.5,
                "num_predict": 600,
                "top_p": 0.8,
                "top_k": 40
            },
            "stream": False
        }
    )
    return response.json()["response"]

@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def ollama_call_long(system_prompt, user_prompt):
    """Replace bard_call_long with local Ollama model (extended output)."""
    response = requests.post(
        f"{OLLAMA_BASE_URL}/api/generate",
        json={
            "model": "llama3:70b",  # Use larger model for extended context
            "prompt": f"{system_prompt}\n\nUser: {user_prompt}\n\nAssistant:",
            "options": {
                "temperature": 0.4,
                "num_predict": 8000,
                "top_p": 0.8,
                "top_k": 40,
                "num_ctx": 32768  # Extended context window
            },
            "stream": False
        }
    )
    return response.json()["response"]
```

### 7.5 Parameter Mapping Reference

| Parameter | Palm2 | OpenAI | Claude | Mistral | Ollama |
|-----------|-------|--------|--------|---------|--------|
| temperature | 0.5 | 0.5 | 0.5 | 0.5 | 0.5 |
| max_output_tokens | 600 | max_tokens=600 | max_tokens=600 | max_tokens=600 | num_predict=600 |
| top_p | 0.8 | 0.8 | 0.8 | 0.8 | 0.8 |
| top_k | 40 | N/A | N/A | N/A | 40 |

---

## 8. Error Handling Patterns

### 8.1 Retry Configuration

```python
from tenacity import (
    retry,
    wait_random_exponential,
    stop_after_attempt,
    retry_if_exception_type
)

# Standard retry decorator
@retry(
    wait=wait_random_exponential(min=1, max=60),
    stop=stop_after_attempt(6),
    retry=retry_if_exception_type((
        ConnectionError,
        TimeoutError,
        RateLimitError  # Provider-specific
    ))
)
def api_call_with_retry():
    pass
```

### 8.2 Error Categories and Handling

```python
class WorkflowError(Exception):
    """Base exception for workflow errors."""
    pass

class LLMError(WorkflowError):
    """LLM API call failures."""
    pass

class SearchError(WorkflowError):
    """Google Search API failures."""
    pass

class ExtractionError(WorkflowError):
    """Content extraction failures."""
    pass

class ParsingError(WorkflowError):
    """Entity parsing failures (ast.literal_eval)."""
    pass

# Error handling wrapper
def safe_extract_entities(response):
    """Safely extract entities with fallback."""
    try:
        entities = extract_entities_bard(response)
        return entities
    except ParsingError:
        # Fallback: Return empty list
        return []
    except LLMError as e:
        # Log and retry logic handled by decorator
        raise
```

### 8.3 Graceful Degradation

```python
def process_with_fallback(query):
    """Process query with graceful degradation."""
    try:
        # Primary: Full processing
        result = query_bard(query, 0)
    except LLMError:
        # Fallback 1: Use only extractive summary (no LLM)
        results = google_search(query, limit, iso)
        content = ' '.join([r[2] for r in results])  # Use snippets
        summary = extractive_summary(content)
        result = pd.DataFrame({
            'Query': [query],
            'Response': [summary],
            'Entity': ['N/A'],
            'Mentions': [0],
            'Prominence': [0.0]
        })
    except SearchError:
        # Fallback 2: Return empty result
        result = pd.DataFrame(columns=['Query', 'Response', 'Entity', 'Mentions', 'Prominence'])

    return result
```

---

## 9. Prompt Templates

### 9.1 Entity Extraction Prompt

```python
ENTITY_EXTRACTION_SYSTEM = """You are a helpful assistant. Your task is to extract
specific SEO elements such as named entities and domain-specific expressions from a text."""

ENTITY_EXTRACTION_USER = """Can you extract any specific SEO elements from the
following text: "{text}"? Please respond only with a list of items in the
following format, removing stop words and the result is a Python list []."""
```

### 9.2 Query Expansion Prompt

```python
QUERY_EXPANSION_SYSTEM = """You are an SEO expert. Generate related search queries
that people might use when researching the given topic."""

QUERY_EXPANSION_USER = """Generate 10 related search queries for the topic: "{topic}".
Return only a Python list of strings, no explanations."""
```

### 9.3 Topical Map Generation Prompt

```python
TOPICAL_MAP_SYSTEM = "You are a helpful SEO and content strategy assistant."

TOPICAL_MAP_USER = """
Please create a topical map for {topic}, based on the keywords: {keywords}.
Respect the iso code {iso}
Visualize the topical gap in table format.

Step 1:
Your first task is to function as a topical map architect.
Your objective is to compile a comprehensive list of nouns, predicates,
and named entities that semantically relate to a given topic.
Your map should include at least 50 nouns/predicates associated with [Your Topic].
These should be categorized into five primary sections.

Step 2:
Create a content strategy with pillar pages and supporting articles.

Step 3:
Identify content gaps and opportunities.

Step 4:
Provide keyword clustering recommendations.

Step 5:
Generate a prioritized action plan.
"""
```

---

## 10. Dependencies and Installation

### 10.1 Requirements File (requirements.txt)

```
# Core
pandas>=1.5.0
numpy>=1.23.0

# LLM - Choose one based on provider
# google-cloud-aiplatform>=1.25.0  # For Google Vertex AI
# openai>=1.0.0                    # For OpenAI
# anthropic>=0.7.0                 # For Anthropic
# mistralai>=0.0.7                 # For Mistral

# Search
google-api-python-client>=2.80.0

# Web Scraping
trafilatura>=1.6.0

# Text Processing
scikit-learn>=1.2.0
sentence-transformers>=2.2.0

# Fuzzy Matching
fuzzywuzzy>=0.18.0
python-Levenshtein>=0.20.0  # Speeds up fuzzywuzzy

# Retry Logic
tenacity>=8.2.0

# Optional: Progress bars
tqdm>=4.65.0
```

### 10.2 Installation Commands

```bash
# Create virtual environment
python -m venv keyword_research_env
source keyword_research_env/bin/activate  # Linux/Mac
# or
keyword_research_env\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt

# For Google Vertex AI (original)
pip install google-cloud-aiplatform

# For OpenAI replacement
pip install openai

# For Anthropic Claude replacement
pip install anthropic

# For Mistral replacement
pip install mistralai

# For local models (Ollama)
# Install Ollama from https://ollama.ai
# Then pull a model: ollama pull llama3:8b
```

### 10.3 Environment Variables

```bash
# Google Cloud (Original)
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"
export GCP_PROJECT="your-project-id"
export GCP_LOCATION="us-central1"

# Google Custom Search
export GOOGLE_API_KEY="your-api-key"
export CUSTOM_SEARCH_ENGINE_ID="your-cse-id"

# OpenAI (Replacement)
export OPENAI_API_KEY="your-openai-key"

# Anthropic (Replacement)
export ANTHROPIC_API_KEY="your-anthropic-key"

# Mistral (Replacement)
export MISTRAL_API_KEY="your-mistral-key"
```

---

## Appendix A: Complete Workflow Example

```python
"""
Complete Automated Keyword Research Workflow
Replace LLM calls as needed based on your provider.
"""

import pandas as pd
from concurrent.futures import ThreadPoolExecutor
import warnings
warnings.filterwarnings('ignore')

# Configuration
topic = "automated seo tools"
limit = 10
iso = "en"

def run_keyword_research(topic):
    """Execute complete keyword research workflow."""

    print(f"Starting keyword research for: {topic}")

    # Step 1: Generate related queries
    print("Step 1: Generating related queries...")
    queries = generate_related_queries(topic)
    queries.insert(0, topic)  # Include original topic
    print(f"Generated {len(queries)} queries")

    # Step 2-5: Process each query (with parallelization)
    print("Steps 2-5: Processing queries...")
    all_results = []

    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = [executor.submit(query_bard, q, i) for i, q in enumerate(queries)]
        for future in futures:
            result = future.result()
            if not result.empty:
                all_results.append(result)

    # Combine results
    master_df = pd.concat(all_results, ignore_index=True)
    print(f"Collected {len(master_df)} entity records")

    # Step 6: Deduplicate entities
    print("Step 6: Deduplicating entities...")
    unified_df = unify_similar_entity(master_df, threshold=90)

    # Step 7: Analyze entities
    print("Step 7: Analyzing entities...")
    analysis_df = analyze_entity(unified_df)
    analysis_df = analysis_df.sort_values('Total Mentions', ascending=False)
    print(f"Found {len(analysis_df)} unique entities")

    # Step 8: Generate report
    print("Step 8: Generating topical map report...")
    report = generate_report(analysis_df, topic)

    return {
        'raw_data': master_df,
        'analysis': analysis_df,
        'report': report
    }

# Execute
if __name__ == "__main__":
    results = run_keyword_research(topic)

    # Save outputs
    results['raw_data'].to_csv('keyword_raw_data.csv', index=False)
    results['analysis'].to_csv('keyword_analysis.csv', index=False)
    with open('topical_map_report.md', 'w') as f:
        f.write(results['report'])

    print("Workflow complete! Files saved.")
```

---

## Document Metadata

- **Version**: 1.0.0
- **Created**: 2025-01-XX
- **Original Source**: `Data_Marketing_Labs_Large_Language_Model_Search_Optimization_SGE_Full_Palm2_For_Students (6).ipynb`
- **Purpose**: Enable 100% workflow reproduction with any LLM provider
- **Compatibility**: Python 3.8+

---

*This documentation provides complete technical specifications for reproducing the Automated Keyword Research Workflow with any modern LLM technology. All function implementations, parameters, and prompt templates are included to ensure exact replication of the original workflow behavior.*
