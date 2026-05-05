[![MseeP.ai Security Assessment Badge](https://mseep.net/pr/overtlids-mcp-searxng-enhanced-badge.png)](https://mseep.ai/app/overtlids-mcp-searxng-enhanced)

# MCP SearXNG Enhanced Server

A Model Context Protocol (MCP) server for category-aware web search, website scraping, and date/time tools. Designed for seamless integration with SearXNG and modern MCP clients.

## Features

- 🔍 SearXNG-powered web search with category support (general, images, videos, files, map, social media)
- 📄 Website content scraping with citation metadata and automatic Reddit URL conversion
- 📜 Intial PDF reading support with a conversion to Markdown using [PyMuPDF/PyMuPDF4LLM](https://pymupdf.readthedocs.io/en/latest/pymupdf4llm/index.html#pymupdf4llm)
- 💾 In-memory caching with automatic freshness validation
- 🚦 Domain-based rate limiting to prevent service abuse
- 🕒 Timezone-aware date/time tool
- ⚠️ Robust error handling with custom exception types
- 🐳 Dockerized and configurable via environment variables
- ⚙️ Configuration persistence between container restarts

## Quick Start

### Prerequisites

- Docker installed on your system
- A running SearXNG instance (self-hosted or accessible endpoint)

### Installation & Usage

**Build the Docker image:**
```bash
docker build -t overtlids/mcp-searxng-enhanced:latest .
```

**Run with your SearXNG instance (Manual Docker Run):**
```bash
docker run -i --rm --network=host \
  -e SEARXNG_ENGINE_API_BASE_URL="http://127.0.0.1:8080/search" \
  -e DESIRED_TIMEZONE="America/New_York" \
  overtlids/mcp-searxng-enhanced:latest
```
In this example, `SEARXNG_ENGINE_API_BASE_URL` is explicitly set. `DESIRED_TIMEZONE` is also explicitly set to `America/New_York`, which matches its default value. If an environment variable is not provided using an `-e` flag during the `docker run` command, the server will automatically use the default value defined in its `Dockerfile` (refer to the Environment Variables table below). Thus, if you intend to use the default for `DESIRED_TIMEZONE`, you could omit the `-e DESIRED_TIMEZONE="America/New_York"` flag. However, `SEARXNG_ENGINE_API_BASE_URL` is critical and usually needs to be set to match your specific SearXNG instance's address if the Dockerfile default (`http://host.docker.internal:8080/search`) is not appropriate.

**Note on Manual Docker Run:** This command runs the Docker container independently. If you are using an MCP client (like Cline in VS Code) to manage this server, the client will start its own instance of the container using the settings defined in *its own configuration*. For the MCP client to use specific environment variables, they **must** be configured within the client's settings for this server (see below).

**Configure your MCP client** (e.g., Cline in VS Code):

For your MCP client to correctly manage and run this server, you **must** define all necessary environment variables within the client's settings for the `overtlids/mcp-searxng-enhanced` server. The MCP client will use these settings to construct the `docker run` command.

The following is the **recommended default configuration** for this server within your MCP client's JSON settings (e.g., `cline_mcp_settings.json`). This example explicitly lists all environment variables set to their default values as defined in the `Dockerfile`. You can copy and paste this directly and then customize any values as needed.

```json
{
  "mcpServers": {
    "overtlids/mcp-searxng-enhanced": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm", "--network=host",
        "-e", "SEARXNG_ENGINE_API_BASE_URL=http://host.docker.internal:8080/search",
        "-e", "DESIRED_TIMEZONE=America/New_York",
        "-e", "ODS_CONFIG_PATH=/config/ods_config.json",
        "-e", "RETURNED_SCRAPPED_PAGES_NO=3",
        "-e", "SCRAPPED_PAGES_NO=5",
        "-e", "PAGE_CONTENT_WORDS_LIMIT=5000",
        "-e", "CITATION_LINKS=True",
        "-e", "MAX_IMAGE_RESULTS=10",
        "-e", "MAX_VIDEO_RESULTS=10",
        "-e", "MAX_FILE_RESULTS=5",
        "-e", "MAX_MAP_RESULTS=5",
        "-e", "MAX_SOCIAL_RESULTS=5",
        "-e", "TRAFILATURA_TIMEOUT=15",
        "-e", "SCRAPING_TIMEOUT=20",
        "-e", "CACHE_MAXSIZE=100",
        "-e", "CACHE_TTL_MINUTES=5",
        "-e", "CACHE_MAX_AGE_MINUTES=30",
        "-e", "RATE_LIMIT_REQUESTS_PER_MINUTE=10",
        "-e", "RATE_LIMIT_TIMEOUT_SECONDS=60",
        "-e", "IGNORED_WEBSITES=",
        "overtlids/mcp-searxng-enhanced:latest"
      ],
      "timeout": 60
    }
  }
}
```
**Key Points for MCP Client Configuration:**
- The example above provides a complete set of arguments to run the Docker container with all environment variables set to their default values.
- To customize any setting, simply modify the value for the corresponding `-e "VARIABLE_NAME=value"` line within the `args` array in your MCP client's configuration. For instance, to change `SEARXNG_ENGINE_API_BASE_URL` and `DESIRED_TIMEZONE`, you would adjust their respective lines.
- Refer to the "Environment Variables" table below for a detailed description of each variable and its default.
- The server's behavior is primarily controlled by these environment variables. While an `ods_config.json` file can also influence settings (see Configuration Management), environment variables passed by the MCP client take precedence.

## HTTP Server Mode (FastMCP)

In addition to the default stdio transport, the server can expose an MCP endpoint over HTTP using [FastMCP](https://github.com/jlowin/fastmcp). This is useful for clients that connect via HTTP rather than spawning a subprocess.

### Starting the HTTP Server

```bash
python mcp_server.py --http
```

The server will start on `0.0.0.0:8000` by default and accept MCP requests at:

```
http://<host>:<port>/mcp
```

All origins are allowed (CORS is fully open), so the endpoint is accessible from any client or browser-based tool.

### HTTP Server Environment Variables

| Variable         | Description                        | Default     |
|------------------|------------------------------------|-------------|
| `MCP_HTTP_HOST`  | Host address to bind to            | `0.0.0.0`   |
| `MCP_HTTP_PORT`  | Port to listen on                  | `8000`      |

**Example — custom host and port:**

```bash
MCP_HTTP_HOST=127.0.0.1 MCP_HTTP_PORT=9000 python mcp_server.py --http
```

**Windows (Command Prompt):**
```cmd
set MCP_HTTP_HOST=127.0.0.1
set MCP_HTTP_PORT=9000
python mcp_server.py --http
```

**Windows (PowerShell):**
```powershell
$env:MCP_HTTP_HOST="127.0.0.1"
$env:MCP_HTTP_PORT="9000"
python mcp_server.py --http
```

### Connecting an MCP Client via HTTP

Point your MCP client at the `/mcp` endpoint:

```json
{
  "mcpServers": {
    "mcp-searxng-enhanced-http": {
      "url": "http://localhost:8000/mcp"
    }
  }
}
```

> **Note:** All server configuration variables (`SEARXNG_ENGINE_API_BASE_URL`, `DESIRED_TIMEZONE`, etc.) apply in HTTP mode exactly as they do in stdio mode. The `ods_config.json` file is written at startup before the server begins accepting connections.

---

## Running Natively (Without Docker)

If you prefer to run the server directly using Python without Docker, follow these steps:

**1. Python Installation:**
   - This server requires **Python 3.9 or newer**. Python 3.11 (as used in the Docker image) is recommended.
   - You can download Python from [python.org](https://www.python.org/downloads/).

**2. Clone the Repository:**
   - Get the code from GitHub:
     ```bash
     git clone https://github.com/OvertliDS/mcp-searxng-enhanced.git
     cd mcp-searxng-enhanced
     ```

**3. Create and Activate a Virtual Environment (Recommended):**
   - Using a virtual environment helps manage dependencies and avoid conflicts with other Python projects.
     ```bash
     # For Linux/macOS
     python3 -m venv .venv
     source .venv/bin/activate

     # For Windows (Command Prompt)
     python -m venv .venv
     .\.venv\Scripts\activate.bat

     # For Windows (PowerShell)
     python -m venv .venv
     .\.venv\Scripts\Activate.ps1
     ```

**4. Install Dependencies:**
   - Install the required Python packages:
     ```bash
     pip install -r requirements.txt
     ```
     Key dependencies include `httpx`, `BeautifulSoup4`, `pydantic`, `trafilatura`, `python-dateutil`, `cachetools`, `zoneinfo`, `filetype`, `pymupdf`, `pymupdf4llm`, and `fastmcp`.

**5. Ensure SearXNG is Accessible:**
   - You still need a running SearXNG instance. Make sure you have its API base URL (e.g., `http://127.0.0.1:8080/search`).

**6. Set Environment Variables:**
   - The server is configured via environment variables. At a minimum, you'll likely need to set `SEARXNG_ENGINE_API_BASE_URL`.
   - **Linux/macOS (bash/zsh):**
     ```bash
     export SEARXNG_ENGINE_API_BASE_URL="http://your-searxng-instance:port/search"
     export DESIRED_TIMEZONE="America/Los_Angeles"
     ```
   - **Windows (Command Prompt):**
     ```bash
     set SEARXNG_ENGINE_API_BASE_URL="http://your-searxng-instance:port/search"
     set DESIRED_TIMEZONE="America/Los_Angeles"
     ```
   - **Windows (PowerShell):**
     ```bash
     $env:SEARXNG_ENGINE_API_BASE_URL="http://your-searxng-instance:port/search"
     $env:DESIRED_TIMEZONE="America/Los_Angeles"
     ```
   - Refer to the "Environment Variables" table below for all available options. If not set, defaults from the script or an `ods_config.json` file (if present in the root directory or at `ODS_CONFIG_PATH`) will be used.

**7. Run the Server:**
   - **stdio mode** (default — for MCP clients that spawn a subprocess):
     ```bash
     python mcp_server.py
     ```
     The server listens for MCP client connections via stdin/stdout.

   - **HTTP mode** (for MCP clients that connect via HTTP):
     ```bash
     python mcp_server.py --http
     ```
     The server starts a FastMCP HTTP endpoint at `http://0.0.0.0:8000/mcp`. See [HTTP Server Mode](#http-server-mode-fastmcp) for configuration options.

**8. Configuration File (`ods_config.json`):**
   - Alternatively, or in combination with environment variables, you can create an `ods_config.json` file in the project's root directory (or the path specified by the `ODS_CONFIG_PATH` environment variable). Environment variables will always take precedence over values in this file. Example:
    ```json
    {
      "searxng_engine_api_base_url": "http://127.0.0.1:8080/search",
      "desired_timezone": "America/New_York"
    }
    ```

## Environment Variables

The following environment variables control the server's behavior. You can set them in your MCP client's configuration (recommended for client-managed servers) or when running Docker manually.

| Variable                        | Description                                | Default (from Dockerfile)         | Notes                                                                                                |
|---------------------------------|--------------------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------|
| `SEARXNG_ENGINE_API_BASE_URL`   | SearXNG search endpoint                    | `http://host.docker.internal:8080/search` | **Crucial for server operation**                                                                     |
| `MCP_HTTP_HOST`                 | Bind address for HTTP server mode          | `0.0.0.0`                         | Only used when starting with `--http`                                                               |
| `MCP_HTTP_PORT`                 | Port for HTTP server mode                  | `8000`                            | Only used when starting with `--http`                                                               |
| `DESIRED_TIMEZONE`              | Timezone for date/time tool                | `America/New_York`                | E.g., `America/Los_Angeles`. List of tz database time zones: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones |
| `ODS_CONFIG_PATH`               | Path to persistent configuration file      | `/config/ods_config.json`         | Typically left as default within the container.     |
| `RETURNED_SCRAPPED_PAGES_NO`    | Max pages to return per search             | `3`                               |                                                       |
| `SCRAPPED_PAGES_NO`             | Max pages to attempt scraping              | `5`                               |                                                       |
| `PAGE_CONTENT_WORDS_LIMIT`      | Max words per scraped page                 | `5000`                            |                                                       |
| `CITATION_LINKS`                | Enable/disable citation events             | `True`                            | `True` or `False`                                     |
| `MAX_IMAGE_RESULTS`             | Maximum image results to return            | `10`                              |                                                       |
| `MAX_VIDEO_RESULTS`             | Maximum video results to return            | `10`                              |                                                       |
| `MAX_FILE_RESULTS`              | Maximum file results to return             | `5`                               |                                                       |
| `MAX_MAP_RESULTS`               | Maximum map results to return              | `5`                               |                                                       |
| `MAX_SOCIAL_RESULTS`            | Maximum social media results to return     | `5`                               |                                                       |
| `TRAFILATURA_TIMEOUT`           | Content extraction timeout (seconds)       | `15`                              |                                                       |
| `SCRAPING_TIMEOUT`              | HTTP request timeout (seconds)             | `20`                              |                                                       |
| `CACHE_MAXSIZE`                 | Maximum number of cached websites          | `100`                             |                                                       |
| `CACHE_TTL_MINUTES`             | Cache time-to-live (minutes)               | `5`                               |                                                       |
| `CACHE_MAX_AGE_MINUTES`         | Maximum age for cached content (minutes)   | `30`                              |                                                       |
| `RATE_LIMIT_REQUESTS_PER_MINUTE`| Max requests per domain per minute         | `10`                              |                                                       |
| `RATE_LIMIT_TIMEOUT_SECONDS`    | Rate limit tracking window (seconds)       | `60`                              |                                                       |
| `IGNORED_WEBSITES`              | Comma-separated list of sites to ignore    | `""` (empty)                      | E.g., `"example.com,another.org"`                   |

## Configuration Management

The server uses a three-tier configuration approach:

1. **Script defaults** (hardcoded in Python)
2. **Config file** (loaded from `ODS_CONFIG_PATH`, defaults to `/config/ods_config.json`)
3. **Environment variables** (highest precedence)

The config file is only updated when:
- The file doesn't exist yet (first-time initialization)
- Environment variables are explicitly provided for the current run

This ensures that user configurations are preserved between container restarts when no new environment variables are set.

## Tools & Aliases

| Tool Name              | Purpose                       | Aliases                           |
|------------------------|------------------------------ |-----------------------------------|
| `search_web`           | Web search via SearXNG        | `search`, `web_search`, `find`, `lookup_web`, `search_online`, `access_internet`, `lookup`* |
| `get_website`          | Scrape website content        | `fetch_url`, `scrape_page`, `get`, `load_website`, `lookup`* |
| `get_current_datetime` | Current date/time             | `current_time`, `get_time`, `current_date` |

\*`lookup` is context-sensitive:  
- If called with a `url` argument, it maps to `get_website`  
- Otherwise, it maps to `search_web`

### Example: Calling Tools

**Web Search**
```json
{ "name": "search_web", "arguments": { "query": "open source ai" } }
```
or using an alias:
```json
{ "name": "search", "arguments": { "query": "open source ai" } }
```

**Category-Specific Search**
```json
{ "name": "search_web", "arguments": { "query": "landscapes", "category": "images" } }
```

**Website Scraping**
```json
{ "name": "get_website", "arguments": { "url": "example.com" } }
```
or using an alias:
```json
{ "name": "lookup", "arguments": { "url": "example.com" } }
```

**Current Date/Time**
```json
{ "name": "get_current_datetime", "arguments": {} }
```
or:
```json
{ "name": "current_time", "arguments": {} }
```

## Advanced Features

### Category-Specific Search

The `search_web` tool supports different categories with tailored outputs:

- **images**: Returns image URLs, titles, and source pages with optional Markdown embedding
- **videos**: Returns video information including titles, source, and embed URLs
- **files**: Returns downloadable file information including format and size
- **map**: Returns location data including coordinates and addresses
- **social media**: Returns posts and profiles from social platforms
- **general**: Default category that scrapes and returns full webpage content

### Reddit URL Conversion

When scraping Reddit content, URLs are automatically converted to use the old.reddit.com domain for better content extraction.

### Rate Limiting

Domain-based rate limiting prevents excessive requests to the same domain within a time window. This prevents overwhelming target websites and potential IP blocking.

### Cache Validation

Cached website content is automatically validated for freshness based on age. Stale content is refreshed automatically while valid cached content is served quickly.

## Error Handling

The server implements a robust error handling system with these exception types:

- `MCPServerError`: Base exception class for all server errors
- `ConfigurationError`: Raised when configuration values are invalid
- `SearXNGConnectionError`: Raised when connection to SearXNG fails
- `WebScrapingError`: Raised when web scraping fails
- `RateLimitExceededError`: Raised when rate limit for a domain is exceeded

Errors are properly propagated to the client with informative messages.

## Troubleshooting

- **Cannot connect to SearXNG**: Ensure your SearXNG instance is running and the `SEARXNG_ENGINE_API_BASE_URL` environment variable points to the correct endpoint.
- **Rate limit errors**: Adjust `RATE_LIMIT_REQUESTS_PER_MINUTE` if you're experiencing too many rate limit errors.
- **Slow content extraction**: Increase `TRAFILATURA_TIMEOUT` to allow more time for content processing on complex pages.
- **Docker networking issues**: If using Docker Desktop on Windows/Mac, `host.docker.internal` should resolve to the host machine. On Linux, you may need to use the host's IP address instead.
- **HTTP mode not reachable**: Ensure no firewall is blocking `MCP_HTTP_PORT` (default `8000`). Set `MCP_HTTP_HOST=0.0.0.0` to bind on all interfaces, or `127.0.0.1` to restrict to localhost only.
- **HTTP mode session ID error**: The server runs in stateless HTTP mode — each POST to `/mcp` is self-contained. If your client requires session-based transport, switch to stdio mode instead.

## Acknowledgements

Inspired by:
- [SearXNG](https://github.com/searxng/searxng) - Privacy-respecting metasearch engine
- [Trafilatura](https://github.com/adbar/trafilatura) - Web scraping tool for text extraction
- [ihor-sokoliuk/mcp-searxng](https://github.com/ihor-sokoliuk/mcp-searxng) - Original MCP server for SearXNG
- [nnaoycurt](https://github.com/nnaoycurt) ([Better Web Search Tool](https://openwebui.com/t/nnaoycurt/web_search))
- [@bwoodruff2021](https://github.com/bwoodruff2021) ([GetTimeDate Tool](https://openwebui.com/t/bwoodruff2021/gettime))

## License

MIT License © 2025 OvertliDS
