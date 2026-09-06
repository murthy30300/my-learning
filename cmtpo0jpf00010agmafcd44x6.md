---
title: "What If Your Food Delivery App Could Just... Understand You?"
seoTitle: "AI-Powered Food Data Platform with Snowflake, dbt & Airflow"
seoDescription: " built an AI food data platform using AWS S3, Snowflake, dbt, Airflow, RAG, and natural-language-to-SQL to turn raw data into actionable insights."
datePublished: 2026-09-06T10:24:48.592Z
cuid: cmtpo0jpf00010agmafcd44x6
slug: ai-powered-food-data-platform-snowflake-dbt-airflow
cover: https://cdn.hashnode.com/uploads/covers/66f590c9120870cbbe3a72f4/5f738313-0e92-4ab0-acaf-eda02ae31464.png
tags: snowflake, artificial-intelligence, airflow, data-engineering, dbt

---

A few weeks ago I was doing what most of us do at 9 PM — scrolling through a food delivery app, trying to decide what to order.

Filters. Ratings. Endless scrolling through the same restaurants.

At some point I caught myself thinking:

> **Why am I doing all this manually? Why can't I just type “something light, not too spicy, and fast” and have the app actually understand that?**

That simple thought turned into a bigger question:

**What does it actually take, under the hood, to make a food platform understand its data well enough to support an experience like that?**

Not just the chatbot UI.

The actual data plumbing behind it.

Where does the data live? How does raw order data become something reliable and queryable? How do you make customer reviews useful to an AI system? And how do you let someone ask a question in plain English and get an answer from the warehouse without writing SQL?

So instead of just wondering, I decided to build the platform underneath that experience.

* * *

## The Problem

Modern food-delivery platforms generate huge amounts of data:

*   Orders
    
*   Restaurants
    
*   Customers
    
*   Payments
    
*   Delivery performance
    
*   Order items
    
*   Customer reviews
    

But most users never interact with that data directly.

They interact with:

*   Filters
    
*   Search boxes
    
*   Dashboards
    
*   Predefined reports
    
*   Manual analysis
    

A customer might want to ask:

> “What should I order tonight?”

A business user might want to know:

> “Which city has the highest cancellation rate?”

Or:

> “What are customers complaining about most in Hyderabad?”

Answering those questions reliably requires much more than a chatbot.

It requires a data platform that can take messy raw data, transform it into trustworthy models, orchestrate the workflow, and make that information available to AI applications.

That is the problem I wanted to explore.

* * *

## What I Built

I built a production-style food-delivery data platform that takes raw data all the way from ingestion to AI-powered applications.

The project handles:

*   **10M+ orders**
    
*   **~23M order items**
    
*   **300K customer reviews**
    
*   Multiple dimensions such as restaurants, customers, food and menu data
    

The overall flow is:

```text
Raw Data
   ↓
Amazon S3
   ↓
Snowflake RAW
   ↓
dbt STAGING
   ↓
dbt MARTS
   ↓
Airflow Orchestration
   ↓
AI Layer
   ├── Review Enrichment
   ├── RAG Chat
   └── Text-to-SQL
```

The goal wasn't just to build a dashboard.

The goal was to build a **data foundation that AI applications could actually use**.

* * *

## How I Built It

### 1\. Data Lands in Amazon S3

The source layer contains food-delivery style datasets covering restaurants, users, food, menu, orders, order items and reviews.

The files are organized in S3 under:

```text
raw/restaurants/
raw/users/
raw/food/
raw/menu/
raw/orders/
raw/order_items/
raw/reviews/
```

* * *

### 2\. S3 → Snowflake

Snowflake reads the raw data from S3 through a secure storage integration.

The ingestion layer uses:

*   External stage
    
*   CSV file format
    
*   `COPY INTO`
    
*   IAM-based access without storing AWS access keys in the application
    

The raw data lands in:

```text
ZOMATO.RAW
```

This becomes the Bronze layer.

* * *

### 3\. dbt: Raw → Clean → Business Ready

The transformation layer follows a medallion-style approach.

#### Silver: STAGING

dbt staging models clean and standardize the source data.

Examples include:

*   Parsing messy fields
    
*   Converting types
    
*   Renaming columns
    
*   Handling nulls
    
*   Deriving useful fields such as delivery status
    

These models live in:

```text
ZOMATO.STAGING
```

#### Gold: MARTS

The Gold layer contains:

*   Dimensions
    
*   Incremental fact models
    
*   Business marts
    
*   Snapshot/SCD2 models
    

The incremental facts use a MERGE-based strategy so the entire 10M+ order dataset doesn't need to be rebuilt every time the pipeline runs.

The business marts are designed around analytical questions such as:

*   Daily city revenue
    
*   Restaurant performance
    
*   Delivery SLA
    
*   Review insights
    

* * *

### 4\. Airflow Orchestrates Everything

Instead of manually running each step, Airflow coordinates the daily pipeline.

The flow is:

```plaintext
reload_raw
      ↓
dbt_build_core
      ↓
enrich_reviews
      ↓
dbt_build_ai
```

This makes the pipeline repeatable and automated instead of a collection of disconnected scripts.

* * *

## The AI Layer

This is where the original “what if I could just ask?” idea comes in.

I built three AI capabilities on top of the data platform.

### 1\. AI Review Enrichment

Customer reviews are unstructured text.

A review such as:

> “The delivery was late and the food arrived cold.”

contains useful business information, but a normal SQL query can't directly reason about the meaning of that sentence.

The enrichment step uses an LLM to turn review text into structured information such as:

*   Sentiment
    
*   Topic
    
*   Underlying issue
    

That enriched data can then be stored in Snowflake and used by downstream analytics.

So instead of treating reviews as just text, they become another source of structured business signals.

* * *

### 2\. RAG: Chat With Customer Reviews

The next step is letting users ask questions about customer feedback.

For example:

> “What do customers complain about most in Hyderabad?”

A retrieval-based system finds the most relevant reviews and uses those reviews as evidence for the answer.

The important part here is that the answer is grounded in actual customer feedback rather than relying only on what the language model already knows.

* * *

### 3\. Text-to-SQL: Chat With the Warehouse

This is probably the closest implementation of the original idea.

Instead of writing SQL, a user can ask:

> “Which cuisine gets the most orders?”

The system:

```text
Natural Language
       ↓
LLM
       ↓
SQL
       ↓
Safety Check
       ↓
Snowflake
       ↓
Answer
```

For example:

```sql
User:
Which cuisine has the most orders?

        ↓

Generated SQL:
SELECT cuisine,
       COUNT(*) AS orders
FROM FCT_ORDERS
GROUP BY cuisine
ORDER BY orders DESC
LIMIT 1

        ↓

Snowflake
        ↓

Result
```

The generated SQL is checked to make sure it is read-only before it is executed.

This makes the warehouse accessible through natural language while still keeping the execution layer controlled.

* * *

## Why This Project Mattered to Me

I'm a final-year CS student interning as a backend engineer, and much of my hands-on experience so far has been on the application side.

This project was my way of going deeper into data engineering.

I wanted to understand what happens behind the scenes when a system has to deal with:

*   Millions of records
    
*   Incremental processing
    
*   Data quality
    
*   Data modeling
    
*   Cloud storage
    
*   Orchestration
    
*   Unstructured text
    
*   AI applications
    

One of the biggest lessons for me was that a good AI experience depends heavily on everything underneath it.

The chatbot may be the part users see.

But underneath it are:

**data ingestion → modeling → transformation → orchestration → security → reliable querying**

The AI becomes useful only when that foundation is trustworthy.

* * *

## What This Project Actually Demonstrates

The project isn't just about putting an LLM on top of a database.

It demonstrates how a modern data platform can become the foundation for multiple AI experiences.

The same underlying warehouse can support:

```text
Structured Data
      ├── Analytics
      ├── Business Metrics
      ├── AI Enrichment
      ├── RAG
      └── Natural-Language SQL
```

That's the part I found most interesting.

The AI layer isn't replacing the data platform.

**It's making the data platform easier to interact with.**

* * *

## What's Next

I'm continuing to build this out by exploring how far the “just ask it” experience can go.

Some of the areas I'm interested in next are:

*   Scaling AI enrichment to the full review dataset
    
*   Improving the reliability of natural-language SQL generation
    
*   Building richer customer-facing recommendation flows
    
*   Exploring more real-time use cases on top of the warehouse
    

The original question started with something simple:

> **“What should I order tonight?”**

The engineering challenge underneath it turned out to be much bigger.

And that's what made the project worth building.

* * *

## Tech Stack

**Cloud & Storage**

*   Amazon S3
    
*   AWS IAM
    

**Data Warehouse**

*   Snowflake
    

**Data Transformation**

*   dbt
    

**Orchestration**

*   Apache Airflow
    

**AI**

*   Large Language Models
    
*   RAG
    
*   Text-to-SQL
    
*   LLM-based review enrichment
    

**Application**

*   Python
    
*   Streamlit
    

Github : https://github.com/murthy30300/zomato-data-analytics

* * *

#DataEngineering #Snowflake #AI #Airflow #dbt #AWS #LLM #RAG #TextToSQL #BuildInPublic