# How Fenic's Session Catalog Works

This document explains the architecture and implementation of Fenic's session catalog system, which manages databases, tables, views, and MCP tools.

## Architecture Overview

Fenic's catalog uses a **three-layer architecture**:

1. **API Layer** (`fenic.api.catalog.Catalog`) - Public user-facing interface
2. **Interface Layer** (`fenic.core._interfaces.catalog.BaseCatalog`) - Abstract base class
3. **Backend Implementation** (`fenic._backends.local.catalog.LocalCatalog`) - Concrete implementations

```
Session
  └── SessionState
      └── BaseCatalog (interface)
          └── LocalCatalog (local backend)
          └── CloudCatalog (cloud backend)
```

**Location**: `src/fenic/api/session/session.py:130`

## API Layer: User-Facing Catalog

The `Catalog` class (`src/fenic/api/catalog.py:13-774`) is a thin wrapper that:

- Provides a clean, documented public API
- Validates inputs using Pydantic (`@validate_call`)
- Delegates all operations to the underlying `BaseCatalog` implementation

### Key Features

```python
class Catalog:
    def __init__(self, catalog: BaseCatalog):
        self.catalog = catalog  # Wraps backend implementation
```

**Delegation Pattern** (all methods delegate to backend):
```python
def create_table(self, table_name: str, schema: Schema, ...) -> bool:
    return self.catalog.create_table(table_name, schema, ...)
```

**Location**: `src/fenic/api/catalog.py:86-92`

## Interface Layer: BaseCatalog Abstract Class

The `BaseCatalog` (`src/fenic/core/_interfaces/catalog.py:13-174`) defines the contract that all catalog backends must implement.

### Catalog Hierarchy

Fenic supports a **three-level namespace hierarchy**:

```
Catalog (e.g., "my_catalog")
  └── Database/Schema (e.g., "my_database")
      └── Tables, Views, Tools
```

### Core Operations

The interface defines operations for:

1. **Catalog Management** (lines 15-33)
   - `does_catalog_exist()`, `get_current_catalog()`, `set_current_catalog()`
   - `list_catalogs()`, `create_catalog()`, `drop_catalog()`

2. **Database Management** (lines 36-71)
   - `does_database_exist()`, `get_current_database()`, `set_current_database()`
   - `list_databases()`, `create_database()`, `drop_database()`

3. **Table Management** (lines 74-104)
   - `does_table_exist()`, `list_tables()`, `describe_table()`
   - `create_table()`, `drop_table()`, `set_table_description()`

4. **View Management** (lines 106-145)
   - `create_view()`, `drop_view()`, `get_view_plan()`
   - `list_views()`, `does_view_exist()`, `describe_view()`

5. **Tool Management** (lines 147-173)
   - `create_tool()`, `drop_tool()`, `describe_tool()`, `list_tools()`

**Location**: `src/fenic/core/_interfaces/catalog.py:13-174`

## Backend Implementation: LocalCatalog

The `LocalCatalog` (`src/fenic/_backends/local/catalog.py:84-650`) implements `BaseCatalog` using **DuckDB** as the storage backend.

### Storage Architecture

```
DuckDB Database
├── typedef_default (default catalog)
│   ├── typedef_default (default database/schema)
│   │   ├── user_table1
│   │   ├── user_table2
│   │   └── ...
│   ├── my_database
│   │   └── ...
│   ├── __fenic_system (system metadata)
│   │   ├── table_schemas (table metadata)
│   │   ├── table_views (view definitions)
│   │   └── mcp_tools (tool definitions)
│   └── fenic_system (read-only system tables)
│       └── query_metrics (execution metrics)
```

### Key Implementation Details

#### Thread Safety (lines 89-96)

```python
class LocalCatalog(BaseCatalog):
    def __init__(self, connection: duckdb.DuckDBPyConnection):
        self.db_conn = connection
        self.lock = threading.RLock()  # Protects write operations
        self.current_database = DEFAULT_DATABASE_NAME
        self.system_tables = SystemTableClient(self.db_conn.cursor())
```

**Thread Safety Strategy**:
- **Write operations** (create/drop/update) use locks to prevent race conditions
- **Read operations** do NOT use locks (DuckDB handles via MVCC)
- Each thread must use its own cursor

**Location**: `src/fenic/_backends/local/catalog.py:99-104`

#### Catalog Operations (lines 106-138)

Local mode only supports a **single catalog** (`typedef_default`):

```python
def get_current_catalog(self) -> str:
    return DEFAULT_CATALOG_NAME  # Always "typedef_default"

def create_catalog(self, catalog_name: str, ...) -> bool:
    raise CatalogError("Catalog creation not supported in local mode")
```

**Location**: `src/fenic/_backends/local/catalog.py:110-130`

#### Database Operations (lines 140-215)

Databases map to **DuckDB schemas**:

```python
def create_database(self, database_name: str, ...) -> bool:
    with self.lock:  # Write operation requires lock
        cursor = self.db_conn.cursor()
        cursor.execute(f'CREATE SCHEMA IF NOT EXISTS "{db_identifier.db}";')
        return True
```

**Location**: `src/fenic/_backends/local/catalog.py:155-172`

#### Table Operations (lines 237-318)

Tables are stored with:
1. **Physical table** in DuckDB
2. **Metadata** in system table (schema + description)

```python
def create_table(self, table_name: str, schema: Schema, ...) -> bool:
    with self.lock:
        # 1. Create empty DuckDB table
        cursor.execute(
            f"CREATE TABLE IF NOT EXISTS {qualified_name} AS SELECT * FROM temp WHERE 1=0"
        )
        # 2. Save metadata to system table
        self.system_tables.save_table(
            cursor, db_name, table_name, schema, description
        )
        return True
```

**Location**: `src/fenic/_backends/local/catalog.py:416-459`

#### View Operations (lines 338-414)

Views store **logical plans**, not SQL:

```python
def create_view(self, view_name: str, logical_plan: LogicalPlan, ...) -> bool:
    with self.lock:
        # Save the logical plan to system table
        self.system_tables.save_view(
            cursor, db_name, view_name, logical_plan, description
        )
        return True
```

Views are **late-bound** - the logical plan is stored and executed when referenced.

**Location**: `src/fenic/_backends/local/catalog.py:461-491`

#### Tool Operations (lines 512-553)

Tools are MCP-compatible parameterized queries:

```python
def create_tool(
    self,
    tool_name: str,
    tool_description: str,
    tool_params: List[ToolParam],
    tool_query: LogicalPlan,
    result_limit: int = 50,
    ...
) -> bool:
    # 1. Bind and validate tool
    tool_definition = bind_tool(
        tool_name, tool_description, tool_params, result_limit, tool_query
    )
    # 2. Save to system table
    self.system_tables.save_tool(cursor, tool_definition)
    return True
```

**Location**: `src/fenic/_backends/local/catalog.py:520-538`

## System Table Client: Metadata Persistence

The `SystemTableClient` (`src/fenic/_backends/local/system_table_client.py`) manages metadata storage in DuckDB.

### System Tables

1. **`__fenic_system.table_schemas`** (lines 34, 63-105)
   - Stores table schemas and descriptions
   - Schema: `(database_name, table_name, schema_blob, description)`
   - `schema_blob`: Serialized Fenic Schema (supports custom types)

2. **`__fenic_system.table_views`** (lines 35, 59)
   - Stores view logical plans and descriptions
   - Schema: `(database_name, view_name, plan_blob, description)`
   - `plan_blob`: Serialized LogicalPlan

3. **`__fenic_system.mcp_tools`** (lines 36, 60, 813-835)
   - Stores MCP tool definitions
   - Schema: `(tool_name, tool_blob)`
   - `tool_blob`: Serialized UserDefinedTool (protobuf)

4. **`fenic_system.query_metrics`** (read-only, lines 39-40)
   - Query execution metrics
   - Populated automatically during query execution

### Tool Persistence

#### Saving Tools (lines 469-488)

```python
def save_tool(self, cursor: duckdb.DuckDBPyConnection, tool: UserDefinedTool):
    # Serialize tool to protobuf
    tool_proto = self.serde_context.serialize_tool_definition(tool)
    tool_blob = base64.b64encode(tool_proto.SerializeToString())

    # Store in system table
    cursor.execute(
        f'INSERT OR REPLACE INTO "{SYSTEM_SCHEMA_NAME}"."{TOOLS_METADATA_TABLE}"'
        ' (tool_name, tool_blob) VALUES (?, ?)',
        (tool.name, tool_blob)
    )
```

#### Loading Tools (lines 490-510)

```python
def describe_tool(self, cursor: duckdb.DuckDBPyConnection, tool_name: str):
    result = cursor.execute(
        f'SELECT tool_blob FROM "{SYSTEM_SCHEMA_NAME}"."{TOOLS_METADATA_TABLE}"'
        ' WHERE tool_name = ?',
        (tool_name,)
    ).fetchone()

    if result is None:
        return None

    # Deserialize from protobuf
    return self._deserialize_and_resolve_tool(result)
```

#### Listing Tools (lines 512-528)

```python
def list_tools(self, cursor: duckdb.DuckDBPyConnection) -> List[UserDefinedTool]:
    result = cursor.execute(
        f'SELECT tool_blob FROM "{SYSTEM_SCHEMA_NAME}"."{TOOLS_METADATA_TABLE}"'
    ).fetchall()

    return [self._deserialize_and_resolve_tool(row) for row in result]
```

## Session Integration

The `Session` class exposes the catalog through a property:

```python
class Session:
    @property
    def catalog(self) -> Catalog:
        return Catalog(self._session_state.catalog)
```

**Location**: `src/fenic/api/session/session.py:130`

### Usage Flow

```python
# 1. Create session
session = fc.Session.get_or_create(config)

# 2. Access catalog
session.catalog.create_database("my_db")
session.catalog.set_current_database("my_db")

# 3. Create table
df = session.create_dataframe({"id": [1, 2, 3]})
df.write.save_as_table("my_table")

# 4. Create tool
tool_query = session.table("my_table").filter(
    fc.col("id") == fc.tool_param("id", IntegerType)
)
session.catalog.create_tool(
    tool_name="find_by_id",
    tool_description="Find row by ID",
    tool_query=tool_query,
    tool_params=[ToolParam(name="id", description="Row ID")],
)

# 5. List tools (for MCP server)
tools = session.catalog.list_tools()
```

## Key Design Patterns

### 1. **Delegation Pattern**

API layer delegates to backend:
```
Catalog (API) → BaseCatalog (interface) → LocalCatalog (implementation)
```

### 2. **Three-Level Namespace**

```
Catalog → Database → Tables/Views/Tools
```

In local mode, catalog is fixed, but cloud mode supports multiple catalogs.

### 3. **Metadata Separation**

- **Physical storage**: DuckDB tables/schemas
- **Logical metadata**: System tables with serialized schemas/plans

This allows Fenic to support:
- Custom data types (Markdown, Embeddings, etc.)
- Logical plan persistence (views)
- Parameterized queries (tools)

### 4. **Transaction Safety**

Write operations use `DuckDBTransaction` context manager:

```python
with DuckDBTransaction(cursor):
    cursor.execute("CREATE TABLE ...")
    self.system_tables.save_table(...)
    # Auto-commits on success, rolls back on error
```

**Location**: `src/fenic/_backends/local/catalog.py:56-82`

### 5. **Thread Safety**

- Write operations protected by `threading.RLock()`
- Read operations rely on DuckDB's MVCC
- Each thread uses its own cursor

## Persistence

### Local Mode

All catalog data persists in the DuckDB database file specified in `SessionConfig`:

```python
config = fc.SessionConfig(
    app_name="my_app",
    work_dir="~/.fenic"  # DuckDB file stored here
)
```

**Database location**: `~/.fenic/my_app.duckdb`

### Catalog Contents

- **User tables**: Actual data in DuckDB tables
- **Views**: Logical plans in `__fenic_system.table_views`
- **Tools**: Serialized definitions in `__fenic_system.mcp_tools`
- **Schemas**: Type metadata in `__fenic_system.table_schemas`
- **Metrics**: Query metrics in `fenic_system.query_metrics`

## Summary

Fenic's catalog system provides:

1. **Unified namespace** for managing databases, tables, views, and tools
2. **Persistent storage** via DuckDB with metadata in system tables
3. **Custom type support** through schema serialization
4. **Logical plan persistence** for views and tools
5. **MCP integration** through tool definitions
6. **Thread-safe operations** with appropriate locking
7. **Backend abstraction** supporting both local and cloud execution

The catalog is the central coordination point for all data management operations in Fenic, enabling features like MCP server creation, view materialization, and metrics tracking.
