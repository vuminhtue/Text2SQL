# Text-to-SQL Workshop: Asking a Database Questions in Plain English

## Overview

This notebook is a hands-on workshop that builds a **text-to-SQL agent**: a program that takes a question written in everyday language (for example, *"How many employees work at our customers in Dallas?"*), turns it into a database query, runs the query, and explains the result in a sentence.

The agent is built with the [LangChain](https://python.langchain.com) library and uses an open-weight large language model (LLM), **Qwen3 14B**, running locally through [Ollama](https://ollama.com). Because the model runs on our own hardware, the questions and the data never leave the system.

The workshop is written for beginners. Basic Python is enough; no prior SQL experience is assumed.

## Learning Goals

By the end of the workshop you will be able to:

1. Explain what a relational database and SQL are, at the level needed to follow along.
2. Explore an unfamiliar database from Python.
3. Describe how an LLM can act as an *agent* that uses tools.
4. Build a text-to-SQL agent with LangChain and a locally hosted LLM.
5. Check whether the agent's answer is correct, and recognize where these systems go wrong.

## Key Ideas in Brief

**Relational database.** Data is stored in tables. Each row is one record and each column is one attribute. Tables are linked through keys (for example, `deals.account_id` points to a row in `accounts`).

**SQL.** The standard language for asking a relational database questions, for example:

```sql
SELECT SUM(employees_count)
FROM accounts
WHERE billing_city = 'Dallas';
```

**LLM agent.** An LLM placed in a loop where it can call **tools** (ordinary Python functions) and read their output. This notebook uses the **ReAct** ("Reason + Act") pattern:

```
Question
  -> Thought:      what should I do next?
  -> Action:       call a tool, e.g. list the tables
  -> Observation:  read what the tool returned
  -> ... repeat as needed ...
  -> Final Answer
```

## Dataset: `CRM.db`

`CRM.db` is a **synthetic** Customer Relationship Management database (the kind of system a sales team uses to track customers), stored in SQLite. It has 36 tables, including:

| Table | What one row represents | Rows |
|---|---|---|
| `accounts` | a customer company (city, revenue, employee count) | 20,000 |
| `deals` | a sales opportunity with an account (amount, stage) | 80,000 |
| `deal_stages` | a step in the sales process, such as `closed_won` | 7 |
| `industry_types` | an industry, such as Software or Healthcare | 15 |
| `users` | an employee of the sales company | |
| `tickets` | a customer support request | |

The SQL dump can be downloaded [here](https://database-sample-website.s3.eu-north-1.amazonaws.com/datasets/crm-sample-database/v2/flagship/1m/sqlite.sql.gz). Convert it into `CRM.db` with:

```bash
gzip -dc sqlite.sql.gz | sqlite3 CRM.db
```

## Requirements

- Python 3.10 with `langchain`, `langchain-community`, `langchain-classic`, and `pandas` (optionally `langchain-ollama`, the newer replacement for the deprecated `Ollama` class).
- Ollama with the `qwen3:14b` model pulled:
  ```bash
  ollama pull qwen3:14b
  ```
- A GPU is strongly recommended for reasonable response times.
- `CRM.db` in the same folder as the notebook.

> **Note on the cluster setup:** Part 2 of the notebook starts Ollama with `module load testing/ollama/...` and reads the host and port from files written by the SMU SuperPOD Ollama module, because GPU nodes are shared between users. On a personal computer, install Ollama, run `ollama serve`, and set:
> ```python
> ollama_url = "http://localhost:11434"
> ```

## Workshop Outline

### Part 1. Background
Databases, SQL, why text-to-SQL is useful, and what an LLM agent is.

### Part 2. Start the language model
Launch the Ollama server and connect to Qwen3 14B with `temperature=0`, so the model always picks its most likely (least random) answer.

### Part 3. Connect to and explore the database
Wrap `CRM.db` with LangChain's `SQLDatabase`, list the tables, read a table's **schema** (column names, types, and links to other tables), and run a few SQL queries by hand. These hand-written queries give a **ground truth** answer to compare the agent against later.

### Part 4. Meet the agent's tools
`SQLDatabaseToolkit` gives the agent four tools:

| Tool | What it does |
|---|---|
| `sql_db_list_tables` | Returns the names of all tables. |
| `sql_db_schema` | Returns the schema and three sample rows for the named tables. |
| `sql_db_query_checker` | Asks the LLM to review a query for common mistakes before it runs. |
| `sql_db_query` | Runs a query and returns the result, or the error message if it fails. |

Because errors are returned as text instead of crashing, the agent can read its mistake and retry with a corrected query.

### Part 5. Build the text-to-SQL agent
Write a **system prompt** that tells the agent to:

- explore the tables and schemas before writing a query;
- compute totals, counts, and averages in SQL rather than by itself;
- return at most 200 rows for queries that list rows;
- say so when the data cannot answer the question;
- never modify the database.

Then create the agent with `create_sql_agent`, using the `ZERO_SHOT_REACT_DESCRIPTION` agent type, `max_iterations=15`, and `return_intermediate_steps=True` so the SQL it ran can be inspected.

### Part 6. Ask questions and verify the answers
Example results from the notebook:

| Question | Agent's answer | Notes |
|---|---|---|
| How many employees in total work at accounts whose billing city is Dallas? | 21,112,950 | Matches the hand-written query from Part 3 |
| Which 3 industries have the most accounts? | Media (5,205), Legal (2,101), Logistics (1,715) | Requires a `JOIN` between `accounts` and `industry_types` |

A helper function `ask(question)` prints the final answer together with every SQL query the agent ran.

### Part 7. Hands-on exercises
Answers are included in the notebook (click **Show answer**).

1. **Counting with a lookup:** How many deals have been won? (7,252)
2. **Adding up values:** What is the total amount in won deals? (530,986,337.33)
3. **Ranking:** Which five billing cities have the most accounts? (New York, Los Angeles, Chicago, Dallas, Houston)
4. **Ambiguous question:** "Who is our best salesperson?" Discuss which meaning of "best" the agent chose.
5. **Safety check:** "Delete all support tickets." The agent should refuse or be blocked.
6. **Optional:** Edit the system prompt and observe how the agent's behavior changes.

### Part 8. Limitations and key takeaways
Summarized in the sections below.

## Limitations

- **Plausible but wrong answers.** A query can run without error yet answer a slightly different question (wrong column, missing join).
- **Ambiguous questions.** Words such as "best", "recent", or "active" must be turned into exact conditions, and the agent may pick the wrong one.
- **Run-to-run variation.** Small wording changes can produce different SQL; smaller models make more mistakes.
- **Large databases.** Real databases can have thousands of tables. Restrict the agent with `SQLDatabase.from_uri(..., include_tables=[...])`.
- **Speed.** Each Thought/Action/Observation step is a full LLM call.

## Good Practices

- Give the agent a **read-only** connection. Prompt rules are a helpful extra layer, never the only protection. For SQLite, a read-only connection can be opened with:
  ```python
  SQLDatabase.from_uri("sqlite:///file:CRM.db?mode=ro&uri=true")
  ```
- Keep the agent's **intermediate steps** and show the SQL to users so the answers can be checked.
- Build a small **test set** of questions with known answers and re-run it whenever the model or prompt changes.
- Prefer a **locally hosted model** when the data is private.

## Key Takeaways

1. A text-to-SQL agent is an LLM in a loop that uses tools to explore the database, write a query, run it, and fix its own errors.
2. The system prompt and the tool descriptions are how you steer the agent's behavior.
3. The agent is a helpful assistant, not an oracle: always verify its SQL and its answers.
