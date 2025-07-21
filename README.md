

# Chat2SQL Agent

## Overview

This project provides a FastAPI-based web service that acts as a "**Chat2SQL Agent**". It accepts natural language questions about data stored in a database, uses Azure OpenAI (specifically configured for models like GPT-4o Mini) to generate a corresponding T-SQL query, executes the query against a configured Microsoft SQL Server database, and then uses Azure OpenAI again to summarize the results in plain English.

The primary goal is to allow users to query database information without needing to write SQL themselves.

## Features

- **Natural Language Querying:** Accepts user questions in plain English.
- **AI-Powered SQL Generation:** Leverages Azure OpenAI to translate questions into T-SQL queries based on a provided database schema.
  - Generates clean, executable SQL queries without comments or formatting
  - Enforces safety rules (SELECT statements only)
  - Uses schema relationships for better query generation
  - Example-based prompting for consistent output format
- **SQL Validation:** Includes a basic validation step for the generated SQL (implementation details in `app/utils/helper.py`).
- **Database Interaction:** Executes the generated T-SQL query against a configured database (implementation details in `app/utils/helper.py`).
- **AI-Powered Summarization:** Uses Azure OpenAI to summarize the raw query results into a user-friendly format.
  - Converts technical SQL results into plain English
  - Focuses on insights rather than technical details
  - Handles empty results gracefully
  - Example-based prompting for consistent summaries
- **RESTful API:** Exposes functionality via a simple `/query/` endpoint.
- **Error Handling:** Includes error handling for API calls, database interactions, and unexpected issues.
- **Logging:** Configured logging to track requests and errors.
- **Metadata Caching:** Implements both in-memory and file-based caching of database metadata for improved performance:
  - In-memory caching using `@lru_cache`
  - File-based caching in `mdmoperational_metadata_cache.json` and ''
  - Includes table schemas, columns, and foreign key relationships
- **Schema Relationships:** Captures and utilizes foreign key relationships between tables for better query generation.

## Example Usage

### SQL Generation Examples

**Good Query:**

```json
{
  "question": "How many users registered in the last month?"
}
```

Response:

```sql
SELECT COUNT(*) FROM dbo.Users WHERE RegistrationDate >= DATEADD(month, -1, GETDATE())
```

**Bad Query (what to avoid):**

```sql
-- This query counts users registered in the last month
SELECT COUNT(*)
FROM dbo.Users
WHERE RegistrationDate >= DATEADD(month, -1, GETDATE());
```

### Result Summary Examples

**Good Summary:**

```json
{
  "columns": ["count"],
  "data": [[42]]
}
```

Response:
"There are 42 users who registered in the last month."

**Bad Summary (what to avoid):**
"The query returned a count of 42 from the Users table where RegistrationDate is within the last month."

## Prerequisites

- Python 3.8+
- `pip` (Python package installer)
- Access to an Azure OpenAI Service:
  - An API Key
  - Your service endpoint URL
  - A deployed model (e.g., `gpt-4o-mini`)
- Access to a Microsoft SQL Server database with:
  - Server hostname
  - Database name
  - Username and password
  - ODBC Driver 18 for SQL Server installed
- Git (for cloning)

## Installation

1.  **Clone the repository:**

    ```bash
    git clone <your-repository-url>
    cd <your-repository-directory> # Should likely be renamed to chat2sql-agent or similar
    ```

2.  **Create and activate a virtual environment (recommended):**

    ```bash
    python -m venv venv
    # On Windows
    # .\venv\Scripts\activate
    # On macOS/Linux
    source venv/bin/activate
    ```

3.  **Install dependencies:**
    ```bash
    pip install -r requirements.txt
    ```

## Configuration

The application relies on environment variables for configuration, particularly for connecting to Azure OpenAI and your database.

Create a `.env` file in the project root directory (where `requirements.txt` is) and add the following variables:

```dotenv
# --- Azure OpenAI Configuration ---
AZURE_OPENAI_API_KEY="<your_azure_openai_api_key>"
AZURE_OPENAI_ENDPOINT="<your_azure_openai_endpoint_url>" # e.g., https://your-service-name.openai.azure.com/
AZURE_OPENAI_API_TYPE="azure" # Usually 'azure'
OPENAI_API_VERSION="2024-02-15-preview" # Or your desired API version
AZURE_MODEL_NAME="gpt-4o-mini" # Or your deployed chat model name for SQL generation/summarization

# --- Database Configuration ---
DB_SERVER="your_db_server" # e.g., your-server.database.windows.net
DB_DATABASE="your_db_name"
DB_USERNAME="your_db_username"
DB_PASSWORD="your_db_password"
```

**Important Notes:**

- The application will create a `db_metadata_cache.json` file to store database schema information.
- To force a refresh of the cached metadata, you can:
  1. Delete the `db_metadata_cache.json` file
  2. Restart the application
- The metadata cache includes:
  - Table schemas and columns
  - Foreign key relationships between tables
  - This information is used to generate more accurate SQL queries

## Running the Application

Once dependencies are installed and the `.env` file is configured, you can run the FastAPI application using Uvicorn:

```bash
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

- `app.main:app`: Tells Uvicorn where to find the FastAPI application instance (`app`) inside the `app/main.py` file.
- `--reload`: Enables auto-reloading for development (the server restarts when code changes). Remove this for production.
- `--host`: Specifies the IP address to bind to.
- `--port`: Specifies the port to listen on.

The API will be available at `http://127.0.0.1:8000`.

## API Endpoint

### Process Query

- **URL:** `/query/`
- **Method:** `POST`
- **Description:** Takes a natural language question, generates SQL, executes it, and returns a summary.
- **Request Body:**
  ```json
  {
    "question": "Your natural language question about the data"
  }
  ```
  _(Defined by `models.schemas.QueryRequest`)_
- **Response Body (Success):**
  ```json
  {
    "response": "The summarized answer in plain English.",
    "sql_query": "The T-SQL query that was generated and executed.",
    "error": null
  }
  ```
  _(Defined by `models.schemas.QueryResponse`)_
- **Response Body (Error):**

  ```json
  {
    "response": "", // Or a specific error message string
    "sql_query": "The SQL query if generated before the error, otherwise null",
    "error": "A description of the error that occurred."
  }
  ```

  _(Also defined by `models.schemas.QueryResponse`, or a standard FastAPI `HTTPException` detail)_

- **Example (`curl`):**
  ```bash
  curl -X POST "http://127.0.0.1:8000/query/" \
       -H "Content-Type: application/json" \
       -d '{
             "question": "How many users registered last month?"
           }'
  ```

## Project Structure (Corrected)

## Testing

The project includes comprehensive unit tests for all major components. To run the tests:

```bash
# Install test dependencies
pip install pytest pytest-asyncio pytest-cov

# Run all tests
pytest

# Run tests with coverage report
pytest --cov=app tests/

# Run specific test file
pytest tests/test_query_controller.py
```

### Test Structure

The test suite is organized as follows:

- `tests/test_query_controller.py`: Tests for the FastAPI endpoints
- `tests/test_query_service.py`: Tests for the query processing service
- `tests/test_prompt_templates.py`: Tests for prompt generation and OpenAI integration
- `tests/test_db_utils.py`: Tests for database utilities
- `tests/conftest.py`: Common test fixtures and setup

Each test file includes:

- Success cases
- Error handling
- Input validation
- Mocked external dependencies
