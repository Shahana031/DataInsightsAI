# DataInsightsAI

**An intelligent natural-language-to-SQL query system.** Ask questions about your business data in plain English, and DataInsightsAI writes the SQL, runs it on a MySQL database, and shows the results as a table with a short written summary.

> Built as an internship project (PIP4004) at Presidency University, School of Computer Science and Engineering, in collaboration with YMinds.ai.

---

## How it works

```
Browser (indexgpt.html)
   │  POST /sendPrompt  { "prompt": "Top 5 employees by sales this year" }
   ▼
FastAPI server (text2sql.py)
   │  adds the question to an OpenAI Assistants thread
   ▼
GPT-4o assistant ──► calls execute_sql_query(query)
   │                     │
   │                     ▼
   │                 MySQL database (text2sql)
   │                     │  rows / error
   │  ◄──────────────────┘  (on error the assistant fixes the query and retries)
   ▼
JSON response  { "messages": [ {type: "table", ...}, {type: "text", ...} ] }
   ▼
Browser shows the table in a side panel ("Artifacts") and the summary in the chat
```

The assistant is told which tables and columns exist and is instructed to generate **read-only** queries, so `INSERT`, `UPDATE`, `DELETE`, `DROP` and similar statements are not generated. It also limits results to 15 rows unless asked otherwise and uses friendly column aliases.

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | HTML, CSS, JavaScript |
| Backend | Python, FastAPI, Uvicorn |
| AI | OpenAI Assistants API (GPT-4o) with function calling |
| Database | MySQL (for example via XAMPP) |

## Repository structure

```
DataInsightsAI/
├── Text2sql/
│   ├── text2sql.py              # FastAPI backend: OpenAI assistant + MySQL execution
│   ├── indexgpt.html            # Main chat UI served at "/"
│   ├── index.html               # Earlier, simpler UI (not served)
│   └── chatbot.code-workspace   # VS Code workspace file
└── REVIEW PPTS/                 # Internship review presentations (Review 0–3)
```

## Getting started

### Prerequisites

- Python 3.9+
- A MySQL server (XAMPP works well for local development)
- An OpenAI API key with access to `gpt-4o`

### 1. Install dependencies

```bash
pip install fastapi uvicorn openai mysql-connector-python pydantic
```

### 2. Create the database

Create a database named `text2sql` with the three tables the assistant knows about. This is an example schema based on the table descriptions in `text2sql.py`; adjust the types to suit your data.

```sql
CREATE DATABASE text2sql;
USE text2sql;

CREATE TABLE employees (
    emp_id        INT PRIMARY KEY,
    emp_name      VARCHAR(100),
    emp_salary    DECIMAL(12, 2),
    department    VARCHAR(50),   -- e.g. sales, accounting, marketing, development
    job_title     VARCHAR(50),   -- e.g. manager, software engineer, accountant
    emp_email     VARCHAR(100),
    joining_date  DATE,
    emp_status    VARCHAR(20)    -- active / inactive
);

CREATE TABLE products (
    product_id     INT PRIMARY KEY,
    product_name   VARCHAR(100),
    product_price  DECIMAL(12, 2),
    in_stock       INT
);

CREATE TABLE sales (
    product_id   INT,
    price        DECIMAL(12, 2),
    lead_id      INT,            -- emp_id of the employee who made the sale
    sale_date    DATE,
    sale_type    VARCHAR(20),    -- e-commerce / on-site
    sale_status  VARCHAR(20),    -- pending / completed / cancelled
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (lead_id)    REFERENCES employees(emp_id)
);
```

Then add some sample rows to each table.

### 3. Configure credentials

Set your OpenAI API key as an environment variable:

```bash
export OPENAI_API_KEY="your-openai-api-key"
```

The MySQL connection details (`host`, `user`, `password`, `database`) are set in the `db_details` dictionary inside `execute_sql_query()` in `Text2sql/text2sql.py`. Update them to match your MySQL setup. Use a MySQL user with **read-only** (`SELECT`) permissions on the `text2sql` database.

### 4. Run the server

Run it from inside the `Text2sql` folder, because the UI file is loaded by a relative path:

```bash
cd Text2sql
python text2sql.py
```

Open <http://localhost:8000> in your browser.

> Tip: sending the message `test` returns built-in sample data, which is handy for checking the UI. The server still needs a valid OpenAI key to start.

## Example questions

- "Show all active employees in the development department"
- "Who are the top 5 employees by total completed sales?"
- "Which products are low on stock?"
- "What was the total e-commerce revenue last month?"
- "How many sales were cancelled this year, grouped by product?"

## API

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Serves the chat UI (`indexgpt.html`) |
| `POST` | `/sendPrompt` | Body `{"prompt": "..."}`; returns `{"messages": [...]}` with `table` and/or `text` items |
| `GET` | `/newConversation` | Deletes the current conversation thread and starts a new one |

## Known limitations

- **OpenAI Assistants API:** the backend uses OpenAI's Assistants API, which OpenAI scheduled for shutdown on 26 August 2026. Running the project now requires migrating to the Responses API.
- **Summaries don't see the data:** after a query runs, the assistant only receives a success flag, not the result rows, so the written summary may not match the table exactly.
- **Query safety:** read-only behaviour is enforced by the assistant's instructions only. Use a read-only MySQL user.
- **Single conversation:** all users share one conversation thread, so the app is meant for single-user, local use.
- **Hard-coded database settings:** MySQL credentials live in the source code rather than in environment variables.

## Future work

- Migrate to the OpenAI Responses API
- Pass query results back to the model for accurate summaries
- Load database settings from environment variables
- Per-user conversation sessions
- Charts and visualisations alongside tables

## References

- Hong et al., *Next-Generation Database Interfaces: A Survey of LLM-based Text-to-SQL*
- Biswal et al., *Text2SQL is Not Enough: Unifying AI and Databases with TAG*
- Tie et al., *Text2SQL Generation Based on Inverse Normalized Schema Linking*
- Li and Nuo, *Enhancing Text2SQL Generation with Syntactic Information and Multi-task Learning*

## Author

**Fatimathul Shahana** — B.Tech CSE (IoT), Presidency University
Internship guide: Mr. Tanveer Ahmed, Assistant Professor, School of Computer Science and Engineering
