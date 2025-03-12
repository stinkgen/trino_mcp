# Trino MCP Server

Model Context Protocol server for Trino, providing AI models with structured access to Trino's distributed SQL query engine.

⚠️ **BETA RELEASE (v0.1.2)** ⚠️  
This project is stabilizing with core features working and tested. Feel free to fork and contribute!

<a href="https://glama.ai/mcp/servers/j0qhebd1qw">
  <img width="380" height="200" src="https://glama.ai/mcp/servers/j0qhebd1qw/badge" alt="Trino Server MCP server" />
</a>

## Features

- ✅ Fixed Docker container API initialization issue! (reliable server initalization)
- ✅ Exposes Trino resources through MCP protocol
- ✅ Enables AI tools to query and analyze data in Trino
- ✅ Provides transport options (STDIO transport works reliably; SSE transport has issues)
- ✅ Fixed catalog handling for proper Trino query execution
- ✅ Both Docker container API and standalone Python API server options

## Quick Start

```bash
# Start the server with docker-compose
docker-compose up -d

# Verify the API is working
curl -X POST "http://localhost:9097/api/query" \
     -H "Content-Type: application/json" \
     -d '{"query": "SELECT 1 AS test"}'
```

Need a non-containerized version? Run the standalone API:

```bash
# Run the standalone API server on port 8008
python llm_trino_api.py
```

## LLM Integration

Want to give an LLM direct access to query your Trino instance? We've created simple tools for that!

### Command-Line LLM Interface

The simplest way to let an LLM query Trino is through our command-line tool:

```bash
# Simple direct query (perfect for LLMs)
python llm_query_trino.py "SELECT * FROM memory.bullshit.real_bullshit_data LIMIT 5"

# Specify a different catalog or schema
python llm_query_trino.py "SELECT * FROM information_schema.tables" memory information_schema
```

### REST API for LLMs

We offer two API options for integration with LLM applications:

#### 1. Docker Container API (Port 9097)

The Docker container exposes a REST API on port 9097:

```bash
# Execute a query against the Docker container API
curl -X POST "http://localhost:9097/api/query" \
     -H "Content-Type: application/json" \
     -d '{"query": "SELECT 1 AS test"}'
```

#### 2. Standalone Python API (Port 8008)

For more flexible deployments, run the standalone API server:

```bash
# Start the API server on port 8008
python llm_trino_api.py
```

This creates endpoints at:
- `GET http://localhost:8008/` - API usage info
- `POST http://localhost:8008/query` - Execute SQL queries

You can then have your LLM make HTTP requests to this endpoint:

```python
# Example code an LLM might generate
import requests

def query_trino(sql_query):
    response = requests.post(
        "http://localhost:8008/query",
        json={"query": sql_query}
    )
    return response.json()

# LLM-generated query
results = query_trino("SELECT job_title, AVG(salary) FROM memory.bullshit.real_bullshit_data GROUP BY job_title ORDER BY AVG(salary) DESC LIMIT 5")
print(results["formatted_results"])
```

This approach allows LLMs to focus on generating SQL, while our tools handle all the MCP protocol complexity!

## Demo and Validation Scripts 🚀

We've created some badass demo scripts that show how AI models can use the MCP protocol to run complex queries against Trino:

### 1. Bullshit Data Generation and Loading

The `tools/create_bullshit_data.py` script generates a dataset of 10,000 employees with ridiculous job titles, inflated salaries, and a "bullshit factor" rating (1-10):

```bash
# Generate the bullshit data
python tools/create_bullshit_data.py

# Load the bullshit data into Trino's memory catalog
python load_bullshit_data.py
```

### 2. Running Complex Queries through MCP

The `test_bullshit_query.py` script demonstrates end-to-end MCP interaction:
- Connects to the MCP server using STDIO transport
- Initializes the protocol following the MCP spec
- Runs a complex SQL query with WHERE, GROUP BY, HAVING, ORDER BY
- Processes and formats the results

```bash
# Run a complex query against the bullshit data through MCP
python test_bullshit_query.py
```

Example output showing top BS jobs with high salaries:
```
🏆 TOP 10 BULLSHIT JOBS (high salary, high BS factor):
----------------------------------------------------------------------------------------------------
JOB_TITLE             | COUNT                | AVG_SALARY           | MAX_SALARY           | AVG_BS_FACTOR        
----------------------------------------------------------------------------------------------------
Advanced Innovation Jedi | 2                    |            241178.50 |            243458.00 |                 7.50
VP of Digital Officer | 1                    |            235384.00 |            235384.00 |                 7.00
Innovation Technical Architect | 1                    |            235210.00 |            235210.00 |                 9.00
...and more!
```

### 3. API Testing

The `test_llm_api.py` script validates the API functionality:

```bash
# Test the Docker container API 
python test_llm_api.py
```

This performs a comprehensive check of:
- API endpoint discovery
- Documentation availability
- Valid query execution
- Error handling for invalid queries

## Usage

```bash
# Start the server with docker-compose
docker-compose up -d
```

The server will be available at:
- Trino: http://localhost:9095
- MCP server: http://localhost:9096
- API server: http://localhost:9097

## Client Connection

✅ **IMPORTANT**: The client scripts run on your local machine (OUTSIDE Docker) and connect TO the Docker containers. The scripts automatically handle this by using docker exec commands. You don't need to be inside the container to use MCP!

Running tests from your local machine:

```bash
# Generate and load data into Trino
python tools/create_bullshit_data.py  # Generates data locally
python load_bullshit_data.py          # Loads data to Trino in Docker

# Run MCP query through Docker
python test_bullshit_query.py         # Queries using MCP in Docker
```

## Transport Options

This server supports two transport methods, but only STDIO is currently reliable:

### STDIO Transport (Recommended and Working)

STDIO transport works reliably and is currently the only recommended method for testing and development:

```bash
# Run with STDIO transport inside the container
docker exec -i trino_mcp_trino-mcp_1 python -m trino_mcp.server --transport stdio --debug --trino-host trino --trino-port 8080 --trino-user trino --trino-catalog memory
```

### SSE Transport (NOT RECOMMENDED - Has Critical Issues)

SSE is the default transport in MCP but has serious issues with the current MCP 1.3.0 version, causing server crashes on client disconnections. **Not recommended for use until these issues are resolved**:

```bash
# NOT RECOMMENDED: Run with SSE transport (crashes on disconnection)
docker exec trino_mcp_trino-mcp_1 python -m trino_mcp.server --transport sse --host 0.0.0.0 --port 8000 --debug
```

## Known Issues and Fixes

### Fixed: Docker Container API Initialization

✅ **FIXED**: We've resolved an issue where the API in the Docker container returned 503 Service Unavailable responses. The problem was with the `app_lifespan` function not properly initializing the `app_context_global` and Trino client connection. The fix ensures that:

1. The Trino client explicitly connects during startup
2. The AppContext global variable is properly initialized
3. Health checks now work correctly

If you encounter 503 errors, check that your container has been rebuilt with the latest code:

```bash
# Rebuild and restart the container with the fix
docker-compose stop trino-mcp
docker-compose rm -f trino-mcp
docker-compose up -d trino-mcp
```

### MCP 1.3.0 SSE Transport Crashes

There's a critical issue with MCP 1.3.0's SSE transport that causes server crashes when clients disconnect. Until a newer MCP version is integrated, use STDIO transport exclusively. The error manifests as:

```
RuntimeError: generator didn't stop after athrow()
anyio.BrokenResourceError
```

### Trino Catalog Handling

We fixed an issue with catalog handling in the Trino client. The original implementation attempted to use `USE catalog` statements, which don't work reliably. The fix directly sets the catalog in the connection parameters.

## Project Structure

This project is organized as follows:

- `src/` - Main source code for the Trino MCP server
- `examples/` - Simple examples showing how to use the server
- `scripts/` - Useful diagnostic and testing scripts
- `tools/` - Utility scripts for data creation and setup
- `tests/` - Automated tests

Key files:
- `llm_trino_api.py` - Standalone API server for LLM integration
- `test_llm_api.py` - Test script for the API server
- `test_mcp_stdio.py` - Main test script using STDIO transport (recommended)
- `test_bullshit_query.py` - Complex query example with bullshit data
- `load_bullshit_data.py` - Script to load generated data into Trino
- `tools/create_bullshit_data.py` - Script to generate hilarious test data
- `run_tests.sh` - Script to run automated tests
- `examples/simple_mcp_query.py` - Simple example to query data using MCP

## Development

**IMPORTANT**: All scripts can be run from your local machine - they'll automatically communicate with the Docker containers via docker exec commands!

```bash
# Install development dependencies
pip install -e ".[dev]"

# Run automated tests 
./run_tests.sh

# Test MCP with STDIO transport (recommended)
python test_mcp_stdio.py

# Simple example query
python examples/simple_mcp_query.py "SELECT 'Hello World' AS message"
```

## Testing

To test that Trino queries are working correctly, use the STDIO transport test script:

```bash
# Recommended test method (STDIO transport)
python test_mcp_stdio.py
```

For more complex testing with the bullshit data:
```bash
# Load and query the bullshit data (shows the full power of Trino MCP!)
python load_bullshit_data.py
python test_bullshit_query.py
```

For testing the LLM API endpoint:
```bash
# Test the Docker container API
python test_llm_api.py 

# Test the standalone API (make sure it's running first)
python llm_trino_api.py
curl -X POST "http://localhost:8008/query" \
     -H "Content-Type: application/json" \
     -d '{"query": "SELECT 1 AS test"}'
```

## How LLMs Can Use This

LLMs can use the Trino MCP server to:

1. **Get Database Schema Information**:
   ```python
   # Example prompt to LLM: "What schemas are available in the memory catalog?"
   # LLM can generate code to query:
   query = "SHOW SCHEMAS FROM memory"
   ```

2. **Run Complex Analytical Queries**:
   ```python
   # Example prompt: "Find the top 5 job titles with highest average salaries"
   # LLM can generate complex SQL:
   query = """
   SELECT 
     job_title, 
     AVG(salary) as avg_salary
   FROM 
     memory.bullshit.real_bullshit_data
   GROUP BY 
     job_title
   ORDER BY 
     avg_salary DESC
   LIMIT 5
   """
   ```

3. **Perform Data Analysis and Present Results**:
   ```python
   # LLM can parse the response, extract insights and present to user:
   "The highest paying job title is 'Advanced Innovation Jedi' with an average salary of $241,178.50"
   ```

### Real LLM Analysis Example: Bullshit Jobs by Company

Here's a real example of what an LLM could produce when asked to "Identify the companies with the most employees in bullshit jobs and create a Mermaid chart":

#### Step 1: LLM generates and runs the query

```sql
SELECT 
  company, 
  COUNT(*) as employee_count, 
  AVG(bullshit_factor) as avg_bs_factor 
FROM 
  memory.bullshit.real_bullshit_data 
WHERE 
  bullshit_factor > 7 
GROUP BY 
  company 
ORDER BY 
  employee_count DESC, 
  avg_bs_factor DESC 
LIMIT 10
```

#### Step 2: LLM gets and analyzes the results

```
COMPANY | EMPLOYEE_COUNT | AVG_BS_FACTOR
----------------------------------------
Unknown Co | 2 | 9.0
BitEdge | 1 | 10.0
CyberWare | 1 | 10.0
BitLink | 1 | 10.0
AlgoMatrix | 1 | 10.0
CryptoHub | 1 | 10.0
BitGrid | 1 | 10.0
MLStream | 1 | 10.0
CloudCube | 1 | 10.0
UltraEdge | 1 | 10.0
```

#### Step 3: LLM generates a Mermaid chart visualization

```mermaid
%%{init: {'theme': 'forest'}}%%
graph LR
    title[Companies with Most Bullshit Jobs]
    style title fill:#333,stroke:#333,stroke-width:1px,color:white,font-weight:bold,font-size:18px

    Companies --> UnknownCo[Unknown Co]
    Companies --> BitEdge[BitEdge]
    Companies --> CyberWare[CyberWare]
    Companies --> BitLink[BitLink]
    Companies --> AlgoMatrix[AlgoMatrix]
    Companies --> CryptoHub[CryptoHub]
    Companies --> BitGrid[BitGrid]
    Companies --> MLStream[MLStream]
    Companies --> CloudCube[CloudCube]
    Companies --> UltraEdge[UltraEdge]
    
    UnknownCo --- Count2[2 employees]
    BitEdge --- Count1a[1 employee]
    CyberWare --- Count1b[1 employee]
    BitLink --- Count1c[1 employee]
    AlgoMatrix --- Count1d[1 employee]
    CryptoHub --- Count1e[1 employee]
    BitGrid --- Count1f[1 employee]
    MLStream --- Count1g[1 employee]
    CloudCube --- Count1h[1 employee]
    UltraEdge --- Count1i[1 employee]
    
    classDef company fill:#ff5733,stroke:#333,stroke-width:1px,color:white,font-weight:bold;
    classDef count fill:#006100,stroke:#333,stroke-width:1px,color:white,font-weight:bold;
    
    class UnknownCo,BitEdge,CyberWare,BitLink,AlgoMatrix,CryptoHub,BitGrid,MLStream,CloudCube,UltraEdge company;
    class Count2,Count1a,Count1b,Count1c,Count1d,Count1e,Count1f,Count1g,Count1h,Count1i count;
```

**Alternative Bar Chart:**

```mermaid
%%{init: {'theme': 'default'}}%%
pie showData
    title Companies with Bullshit Jobs
    "Unknown Co (BS: 9.0)" : 2
    "BitEdge (BS: 10.0)" : 1
    "CyberWare (BS: 10.0)" : 1
    "BitLink (BS: 10.0)" : 1
    "AlgoMatrix (BS: 10.0)" : 1
    "CryptoHub (BS: 10.0)" : 1
    "BitGrid (BS: 10.0)" : 1
    "MLStream (BS: 10.0)" : 1
    "CloudCube (BS: 10.0)" : 1
    "UltraEdge (BS: 10.0)" : 1
```

#### Step 4: LLM provides key insights

The LLM can analyze the data and provide insights:

- "Unknown Co" has the most employees in bullshit roles (2), while all others have just one
- Most companies have achieved a perfect 10.0 bullshit factor score
- Tech-focused companies (BitEdge, CyberWare, etc.) seem to create particularly meaningless roles
- Bullshit roles appear concentrated at executive or specialized position levels

This example demonstrates how an LLM can:
1. Generate appropriate SQL queries based on natural language questions
2. Process and interpret the results from Trino
3. Create visual representations of the data
4. Provide meaningful insights and analysis

## Accessing the API

The Trino MCP server now includes two API options for accessing data:

### 1. Docker Container API (Port 9097)

```python
import requests
import json

# API endpoint (default port 9097 in Docker setup)
api_url = "http://localhost:9097/api/query"

# Define your SQL query
query_data = {
    "query": "SELECT * FROM memory.bullshit.real_bullshit_data LIMIT 5",
    "catalog": "memory",
    "schema": "bullshit"
}

# Send the request
response = requests.post(api_url, json=query_data)
results = response.json()

# Process the results
if results["success"]:
    print(f"Query returned {results['results']['row_count']} rows")
    for row in results['results']['rows']:
        print(row)
else:
    print(f"Query failed: {results['message']}")
```

### 2. Standalone Python API (Port 8008)

```python
# Same code as above, but with different port
api_url = "http://localhost:8008/query"
```

Both APIs offer the following endpoints:
- `GET /api` - API documentation and usage examples
- `POST /api/query` - Execute SQL queries against Trino

These APIs eliminate the need for wrapper scripts and let LLMs query Trino directly using REST calls, making it much simpler to integrate with services like Claude, GPT, and other AI systems.

## Troubleshooting

### API Returns 503 Service Unavailable

If the Docker container API returns 503 errors:

1. Make sure you've rebuilt the container with the latest code:
   ```bash
   docker-compose stop trino-mcp
   docker-compose rm -f trino-mcp
   docker-compose up -d trino-mcp
   ```

2. Check the container logs for errors:
   ```bash
   docker logs trino_mcp_trino-mcp_1
   ```

3. Verify that Trino is running properly:
   ```bash
   curl -s http://localhost:9095/v1/info | jq
   ```

### Port Conflicts with Standalone API

The standalone API defaults to port 8008 to avoid conflicts. If you see an "address already in use" error:

1. Edit `llm_trino_api.py` and change the port number in the last line:
   ```python
   uvicorn.run(app, host="127.0.0.1", port=8008) 
   ```

2. Run with a custom port via command line:
   ```python
   python -c "import llm_trino_api; import uvicorn; uvicorn.run(llm_trino_api.app, host='127.0.0.1', port=8009)"
   ```

## Future Work

This is now in beta with these improvements planned:

- [ ] Integrate with newer MCP versions when available to fix SSE transport issues
- [ ] Add/Validate support for Hive, JDBC, and other connectors
- [ ] Add more comprehensive query validation across different types and complexities
- [ ] Implement support for more data types and advanced Trino features
- [ ] Improve error handling and recovery mechanisms
- [ ] Add user authentication and permission controls
- [ ] Create more comprehensive examples and documentation
- [ ] Develop admin monitoring and management interfaces
- [ ] Add performance metrics and query optimization hints
- [ ] Implement support for long-running queries and result streaming

---

*Developed by Stink Labs, 2025*