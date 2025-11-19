# How Fenic Wraps FastMCP

This document explains how Fenic creates a wrapper around FastMCP to provide a declarative, DataFrame-first API for building MCP (Model Context Protocol) servers.

## Overview

**Fenic does not reimplement MCP from scratch.** Instead, it wraps FastMCP and provides an abstraction layer that converts declarative DataFrame queries into imperative FastMCP tool functions.

## Core Architecture

The `FenicMCPServer` class (defined in `src/fenic/core/mcp/_server.py`) wraps a FastMCP instance:

```python
class FenicMCPServer:
    """Register Fenic tools and serve them via FastMCP."""

    def __init__(self, ...):
        # Import FastMCP (optional dependency)
        from fastmcp import FastMCP
        from mcp.types import ToolAnnotations

        # Create FastMCP instance internally
        self.mcp = FastMCP(self.server_name)  # ← Wraps FastMCP!

        # Register Fenic tools with FastMCP
        for tool in self.user_defined_tools:
            tool_fn = self._build_user_defined_tool(tool)
            self.mcp.tool(...)(tool_fn)  # ← Use FastMCP's @tool decorator
```

**Key location**: `src/fenic/core/mcp/_server.py:58-92`

## Tool Translation Layer

Fenic converts **declarative DataFrame tools** into **imperative FastMCP functions** through the `_build_user_defined_tool()` method:

```python
def _build_user_defined_tool(self, tool_definition: UserDefinedTool):
    """Build a keyword-argument tool function with per-field schema for FastMCP."""

    # Create a Pydantic model from Fenic's ToolParam definitions
    ParamsModel = create_pydantic_model_for_tool(tool_definition)

    async def tool_fn_wrapper(*args, **kwargs) -> MCPResultSet:
        # 1. Extract and validate parameters
        params_obj = ParamsModel.model_validate(payload_only_tool_params)

        # 2. Bind parameters to DataFrame query placeholders
        bound_plan = bind_parameters(
            tool_definition._parameterized_view,  # ← Original DataFrame with fc.tool_param()
            payload,
            tool_definition.params
        )

        # 3. Execute the DataFrame query
        pl_df, metrics = await asyncio.to_thread(
            lambda: self.session_state.execution.collect(bound_plan)
        )

        # 4. Format results for FastMCP
        return self._handle_result_set(bound_plan, pl_df, effective_limit, table_format)

    # 5. Build FastMCP-compatible function signature
    tool_fn_wrapper.__signature__ = inspect.Signature(parameters=params, ...)
    tool_fn_wrapper.__doc__ = tool_definition.description

    return tool_fn_wrapper
```

**Key location**: `src/fenic/core/mcp/_server.py:172-266`

## Example: Documentation Server

The `server.py` example demonstrates this translation in action:

### Step 1: Define Fenic Tool (Declarative)

```python
# Declarative DataFrame with tool_param placeholders
search_query = (
    session.table("api_df")
    .filter(fc.col("name").rlike(fc.tool_param("query", StringType)))  # ← Placeholder
    .select("type", "name", "qualified_name", "docstring")
)

# Register in Fenic catalog
session.catalog.create_tool(
    tool_name="search_fenic_api",
    tool_query=search_query,  # ← DataFrame becomes tool
    tool_params=[ToolParam(name="query", description="Regex pattern")],
)
```

**Location**: `server.py:39-73`

### Step 2: Create MCP Server

```python
server = create_mcp_server(
    session=session,
    user_defined_tools=session.catalog.list_tools(),  # ← Fenic tools
    ...
)
```

**Location**: `server.py:212-226`

### Step 3: Behind the Scenes (Translation)

```python
# FenicMCPServer converts each Fenic tool to a FastMCP tool
for tool in self.user_defined_tools:
    tool_fn = self._build_user_defined_tool(tool)  # ← Convert to function
    self.mcp.tool(  # ← Register with FastMCP
        annotations=ToolAnnotations(...),
        name=to_snake_case(tool.name)
    )(tool_fn)
```

**Location**: `src/fenic/core/mcp/_server.py:93-99`

## Delegation to FastMCP

All server operations delegate to the wrapped FastMCP instance:

```python
async def run_async(self, transport: MCPTransport = "http", **kwargs):
    await self.mcp.run_async(transport=transport, ...)  # ← Delegates to FastMCP

def run(self, transport: MCPTransport = "http", **kwargs):
    self.mcp.run(transport=transport, ...)  # ← Delegates to FastMCP

def http_app(self, **kwargs):
    return self.mcp.http_app(**kwargs)  # ← Delegates to FastMCP
```

**Location**: `src/fenic/core/mcp/_server.py:115-139`

## Key Abstractions

| Layer | Fenic Side | FastMCP Side |
|-------|-----------|--------------|
| **Tool Definition** | DataFrame with `fc.tool_param()` | Async function with kwargs |
| **Parameters** | `ToolParam` objects | Pydantic model fields |
| **Execution** | `bind_parameters()` + `collect()` | Function call |
| **Results** | Polars DataFrame | `MCPResultSet` (JSON) |

## The `fc.tool_param()` Magic

The key to Fenic's declarative approach is the `fc.tool_param()` function, which creates **placeholders** in DataFrame queries:

```python
fc.col("name").rlike(fc.tool_param("query", StringType))
#                    ↑ This becomes an MCP tool parameter
```

When the MCP tool is invoked:
1. Parameter values are received from the client
2. `bind_parameters()` substitutes placeholders with actual values
3. The DataFrame query is executed with bound parameters
4. Results are formatted and returned

## Execution Flow

```
MCP Client Request
    ↓
FastMCP receives request
    ↓
FastMCP calls Fenic tool wrapper function
    ↓
Fenic validates parameters (Pydantic)
    ↓
Fenic binds parameters to DataFrame query
    ↓
Fenic executes query (collect on DataFrame)
    ↓
Fenic formats results as MCPResultSet
    ↓
FastMCP returns response to client
```

## Summary

**Fenic wraps FastMCP** by:

1. **Creating a FastMCP instance internally** (`src/fenic/core/mcp/_server.py:92`)
2. **Converting declarative DataFrame tools → imperative async functions** (`src/fenic/core/mcp/_server.py:172-266`)
3. **Registering these functions with FastMCP** using `@mcp.tool()` (`src/fenic/core/mcp/_server.py:95-99`)
4. **Delegating all server operations** (run, http_app) to FastMCP (`src/fenic/core/mcp/_server.py:115-139`)
5. **Handling tool execution** by binding parameters, running DataFrame queries, and formatting results (`src/fenic/core/mcp/_server.py:198-207`)

This architecture allows Fenic to provide a **declarative, DataFrame-first API** for MCP tools while leveraging FastMCP's protocol implementation and server infrastructure.

## Advantages of This Approach

1. **Declarative**: Tools are defined as data transformations, not imperative code
2. **Type-safe**: Parameters are strongly typed via Fenic's type system
3. **Optimized**: DataFrame queries can be optimized before execution
4. **Reusable**: Same DataFrame logic can be used for tools, reports, or pipelines
5. **Catalog-managed**: Tools are stored and versioned in the session catalog
6. **Leverages FastMCP**: No need to reimplement MCP protocol details

## Optional Dependency

FastMCP is an optional dependency. To use Fenic's MCP server capabilities:

```bash
pip install "fenic[mcp]"
# or
pip install fastmcp
```

Without this dependency, attempting to create an MCP server will raise an `ImportError` with installation instructions.
