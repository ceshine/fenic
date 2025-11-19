# Rust and Python Integration in Fenic

The `fenic` project leverages Rust to implement high-performance, CPU-bound functionalities, integrating them seamlessly into its Python DataFrame library, particularly with Polars. This integration allows `fenic` to offload data-intensive operations to compiled Rust code, benefiting from Rust's speed and memory efficiency while maintaining the flexibility and ease of use of Python.

## Integration Mechanism

The core of the integration relies on a few key components and processes:

1.  **Rust-based Polars Plugin (`polars_plugins`)**:
    *   The Rust code, located in the `rust/` directory, is structured as a Polars plugin.
    *   It utilizes the `pyo3` crate for creating Python bindings and the `pyo3-polars` crate to expose Rust functions as custom Polars expressions.
    *   The `#[polars_expr]` macro from `pyo3-polars` is crucial here, as it marks Rust functions (e.g., `jq_expr` for JSON querying) to be recognized and invoked by the Polars query engine.

2.  **Building with `maturin`**:
    *   The `pyproject.toml` file configures `maturin` as the build backend for the project.
    *   `maturin` is responsible for compiling the Rust code into a Python extension module (typically a `.so` or `.pyd` file). In `fenic`, this compiled module is named `fenic._polars_plugins`.

3.  **Python Namespace Registration**:
    *   On the Python side, `fenic` defines custom namespaces for Polars expressions. A prime example is the `json` namespace, defined in `src/fenic/_backends/local/polars_plugins/json.py`.
    *   By using the `@pl.api.register_expr_namespace("json")` decorator, `fenic` extends Polars to allow expressions like `pl.col("my_json_col").json.some_function()`.

4.  **Connecting Python Calls to Rust Functions**:
    *   Within these Python namespace classes (e.g., the `Json` class), methods (like `jq()`) are implemented.
    *   These methods use Polars' `register_plugin_function` to create a bridge to the compiled Rust code. This function specifies the path to the compiled Rust plugin and the name of the Rust function (e.g., `jq_expr`) to be executed.
    *   When a user calls `df.select(pl.col("data").json.jq("."))`, this mechanism ensures that the corresponding Rust function handles the computation, passing the necessary data and arguments.

5.  **Logical Plan to Physical Plan Execution**:
    *   `fenic` first constructs a logical plan representing the desired data transformations. During this phase, Rust functions might be used for validation (e.g., `py_validate_jq_query` for JQ syntax validation).
    *   When the logical plan is executed, `fenic`'s internal "transpiler" (found in `src/fenic/_backends/local/transpiler/`) converts it into a Polars physical plan.
    *   This conversion translates `fenic`'s abstract operations into concrete Polars expressions, which then trigger the execution of the linked Rust functions, leveraging their performance benefits.

In essence, the Rust code serves as a powerful, low-level backend, providing efficient implementations for specific data manipulation tasks that are then exposed and orchestrated through Python, offering both performance and developer-friendly APIs.
