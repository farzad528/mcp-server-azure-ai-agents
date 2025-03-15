# Azure AI Search MCP Server

A Model Context Protocol (MCP) server that integrates with Azure AI Search, enabling Claude Desktop to query your Azure AI Search indexes using various search methods.

## Overview

This project provides a simple MCP server implementation that connects Claude Desktop to Azure AI Search. It allows Claude to access and search through your indexed data using three powerful search methods.

## Features

- **Keyword Search**: Traditional lexical search for exact matches
- **Vector Search**: Semantic similarity search using vector embeddings
- **Hybrid Search**: Combines keyword and vector approaches for optimal results

## Requirements

- Python 3.10 or higher
- Azure AI Search service with a configured index containing vectorized text data
- Claude Desktop (latest version)
- Windows or macOS (primary instructions for Windows, adaptable for macOS)

## Quick Start

### 1. Installation

```powershell
# Create a project directory
mkdir mcp-server-azure-ai-search
cd mcp-server-azure-ai-search

# Create a .env file for your Azure Search credentials
echo "AZURE_SEARCH_SERVICE_ENDPOINT=https://your-service-name.search.windows.net" > .env
echo "AZURE_SEARCH_INDEX_NAME=your-index-name" >> .env
echo "AZURE_SEARCH_API_KEY=your-api-key" >> .env
```

### 2. Set Up Virtual Environment

```powershell
# Install uv if you don't have it
# (See https://github.com/astral-sh/uv for installation instructions)

# Create and activate a virtual environment
uv venv
.venv\Scripts\activate

# Install dependencies
uv pip install "mcp[cli]" azure-search-documents==11.5.2 azure-identity python-dotenv
```

### 3. Create the Server Script

Create a file named `azure_search_server.py` with the following content:

```python
#!/usr/bin/env python3
"""Azure AI Search MCP Server for Claude Desktop."""

import os
import sys
from dotenv import load_dotenv
from azure.core.credentials import AzureKeyCredential
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizableTextQuery
from mcp.server.fastmcp import FastMCP

# Add startup message
print("Starting Azure AI Search MCP Server...", file=sys.stderr)

# Load environment variables
load_dotenv()
print("Environment variables loaded", file=sys.stderr)

# Create MCP server
mcp = FastMCP(
    "azure-search", 
    description="MCP server for Azure AI Search integration",
    dependencies=["azure-search-documents==11.5.2", "azure-identity", "python-dotenv"]
)
print("MCP server instance created", file=sys.stderr)

class AzureSearchClient:
    """Client for Azure AI Search service."""
    
    def __init__(self):
        """Initialize Azure Search client with credentials from environment variables."""
        print("Initializing Azure Search client...", file=sys.stderr)
        # Load environment variables
        self.endpoint = os.getenv("AZURE_SEARCH_SERVICE_ENDPOINT")
        self.index_name = os.getenv("AZURE_SEARCH_INDEX_NAME")
        api_key = os.getenv("AZURE_SEARCH_API_KEY")
        
        # Validate environment variables
        if not all([self.endpoint, self.index_name, api_key]):
            missing = []
            if not self.endpoint:
                missing.append("AZURE_SEARCH_SERVICE_ENDPOINT")
            if not self.index_name:
                missing.append("AZURE_SEARCH_INDEX_NAME")
            if not api_key:
                missing.append("AZURE_SEARCH_API_KEY")
            error_msg = f"Missing environment variables: {', '.join(missing)}"
            print(f"Error: {error_msg}", file=sys.stderr)
            raise ValueError(error_msg)
        
        # Initialize the search client
        print(f"Connecting to Azure AI Search at {self.endpoint}", file=sys.stderr)
        self.credential = AzureKeyCredential(api_key)
        self.search_client = SearchClient(
            endpoint=self.endpoint,
            index_name=self.index_name,
            credential=self.credential
        )
        print(f"Azure Search client initialized for index: {self.index_name}", file=sys.stderr)
    
    def keyword_search(self, query, top=5):
        """Perform keyword search on the index."""
        print(f"Performing keyword search for: {query}", file=sys.stderr)
        results = self.search_client.search(
            search_text=query,
            top=top,
            select=["title", "chunk"]
        )
        return self._format_results(results)
    
    def vector_search(self, query, top=5, vector_field="text_vector"):
        """Perform vector search on the index."""
        print(f"Performing vector search for: {query}", file=sys.stderr)
        results = self.search_client.search(
            vector_queries=[
                VectorizableTextQuery(
                    text=query,
                    k_nearest_neighbors=50,
                    fields=vector_field
                )
            ],
            top=top,
            select=["title", "chunk"]
        )
        return self._format_results(results)
    
    def hybrid_search(self, query, top=5, vector_field="text_vector"):
        """Perform hybrid search (keyword + vector) on the index."""
        print(f"Performing hybrid search for: {query}", file=sys.stderr)
        results = self.search_client.search(
            search_text=query,
            vector_queries=[
                VectorizableTextQuery(
                    text=query,
                    k_nearest_neighbors=50,
                    fields=vector_field
                )
            ],
            top=top,
            select=["title", "chunk"]
        )
        return self._format_results(results)

    def _format_results(self, results):
        """Format search results for better readability."""
        formatted_results = []
        for result in results:
            item = {
                "title": result.get("title", "Unknown"),
                "content": result.get("chunk", "")[:1000],  # Limit content length
                "score": result.get("@search.score", 0)
            }
            formatted_results.append(item)
        
        print(f"Formatted {len(formatted_results)} search results", file=sys.stderr)
        return formatted_results

# Initialize Azure Search client
try:
    print("Starting initialization of search client...", file=sys.stderr)
    search_client = AzureSearchClient()
    print("Search client initialized successfully", file=sys.stderr)
except Exception as e:
    print(f"Error initializing search client: {str(e)}", file=sys.stderr)
    # Don't exit - we'll handle errors in the tool functions
    search_client = None

def _format_results_as_markdown(results, search_type):
    """Format search results as markdown for better readability."""
    if not results:
        return f"No results found for your query using {search_type}."
    
    markdown = f"## {search_type} Results\n\n"
    
    for i, result in enumerate(results, 1):
        markdown += f"### {i}. {result['title']}\n"
        markdown += f"Score: {result['score']:.2f}\n\n"
        markdown += f"{result['content']}\n\n"
        markdown += "---\n\n"
    
    return markdown

@mcp.tool()
def keyword_search(query: str, top: int = 5) -> str:
    """
    Perform a keyword-based search on the Azure AI Search index.
    
    Args:
        query: The search query text
        top: Maximum number of results to return (default: 5)
    
    Returns:
        Formatted search results
    """
    print(f"Tool called: keyword_search({query}, {top})", file=sys.stderr)
    if search_client is None:
        return "Error: Azure Search client is not initialized. Check server logs for details."
    
    try:
        results = search_client.keyword_search(query, top)
        return _format_results_as_markdown(results, "Keyword Search")
    except Exception as e:
        error_msg = f"Error performing keyword search: {str(e)}"
        print(error_msg, file=sys.stderr)
        return error_msg

@mcp.tool()
def vector_search(query: str, top: int = 5) -> str:
    """
    Perform a vector similarity search on the Azure AI Search index.
    
    Args:
        query: The search query text
        top: Maximum number of results to return (default: 5)
    
    Returns:
        Formatted search results
    """
    print(f"Tool called: vector_search({query}, {top})", file=sys.stderr)
    if search_client is None:
        return "Error: Azure Search client is not initialized. Check server logs for details."
    
    try:
        results = search_client.vector_search(query, top)
        return _format_results_as_markdown(results, "Vector Search")
    except Exception as e:
        error_msg = f"Error performing vector search: {str(e)}"
        print(error_msg, file=sys.stderr)
        return error_msg

@mcp.tool()
def hybrid_search(query: str, top: int = 5) -> str:
    """
    Perform a hybrid search (keyword + vector) on the Azure AI Search index.
    
    Args:
        query: The search query text
        top: Maximum number of results to return (default: 5)
    
    Returns:
        Formatted search results
    """
    print(f"Tool called: hybrid_search({query}, {top})", file=sys.stderr)
    if search_client is None:
        return "Error: Azure Search client is not initialized. Check server logs for details."
    
    try:
        results = search_client.hybrid_search(query, top)
        return _format_results_as_markdown(results, "Hybrid Search")
    except Exception as e:
        error_msg = f"Error performing hybrid search: {str(e)}"
        print(error_msg, file=sys.stderr)
        return error_msg

# Check if tools are registered
tool_count = len(mcp._tools)
print(f"Registered {tool_count} tools: {list(mcp._tools.keys())}", file=sys.stderr)

if __name__ == "__main__":
    # Run the server with stdio transport (default)
    print("Starting MCP server run...", file=sys.stderr)
    mcp.run()
```

### 4. Configure Claude Desktop

1. Open Claude Desktop
2. Click on the Claude menu in the top bar and select "Settings..."
3. Click on "Developer" in the left sidebar
4. Click "Edit Config"
5. Enter the configuration below (adjust paths to match your environment):

```json
{
  "mcpServers": {
    "azure-search": {
      "command": "C:\\path\\to\\mcp-server-azure-ai-search\\.venv\\Scripts\\python.exe",
      "args": [
        "C:\\path\\to\\mcp-server-azure-ai-search\\azure_search_server.py"
      ],
      "env": {
        "AZURE_SEARCH_SERVICE_ENDPOINT": "https://your-service-name.search.windows.net",
        "AZURE_SEARCH_INDEX_NAME": "your-index-name",
        "AZURE_SEARCH_API_KEY": "your-api-key"
      }
    }
  }
}
```

> **Note:** Replace `C:\\path\\to\\mcp-server-azure-ai-search` with the actual path to your project directory, and update the environment variables with your Azure AI Search credentials.

### 5. Test the Server

1. Restart Claude Desktop to load the new configuration
2. Look for the hammer icon in the bottom right of the input field (indicating MCP tools are available)
3. Try asking Claude questions like:
   - "Can you search for information about AI in my Azure Search index?"
   - "Search for vector databases using the vector search tool"
   - "Find information about neural networks using hybrid search"

## Troubleshooting

If the server doesn't appear in Claude Desktop:

1. Check Claude Desktop logs at `%APPDATA%\Claude\logs\mcp*.log`
2. Verify the paths in the configuration are correct
3. Make sure your `.env` file has the correct credentials
4. Test running the server directly: `python azure_search_server.py`

## Customizing Your Server

You can customize the server by:

1. Modifying the search logic in the `AzureSearchClient` class
2. Adding more search tools using the `@mcp.tool()` decorator
3. Changing the response formatting in `_format_results_as_markdown`
4. Adding field filters or additional search parameters

## License

MIT