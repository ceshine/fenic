# Fenic Semantic Classifier: Architecture & Implementation

This document provides a comprehensive technical deep-dive into the architecture of `fenic.semantic.classify`. It traces the lifecycle of a semantic classification request from the high-level user API call down to the low-level LLM interaction, explaining how Fenic achieves a seamless "Polars-like" experience while orchestrating complex AI operations.

## Architectural Overview

Fenic operates on a three-layer architecture designed to separate intent from execution:

1.  **The Logical Layer:** Captures user intent as a "Logical Plan" without executing code.
2.  **The Compiler Layer:** Transpiles the Logical Plan into a physical execution plan backed by Polars.
3.  **The Runtime Layer:** Executes the actual AI operations, managing prompt engineering, LLM communication, and structured output enforcement.

---

## 1. The Logical Layer: Capturing Intent

When a user writes code like `fc.semantic.classify("content", [ ... ])`, they are **not** invoking a function that runs immediately. Instead, they are constructing a node in a **Logical Plan**.

This design pattern, known as "lazy evaluation," allows Fenic to:
*   Validate types and schemas before any expensive API calls are made.
*   Optimize the query plan (though this is more relevant for complex joins).
*   Defer the choice of execution engine (local vs. cloud) until the last possible moment.

The core class representing this specific intent is `SemanticClassifyExpr`. It acts as a frozen blueprint of the user's request.

**File:** `src/fenic/core/_logical_plan/expressions/semantic.py`

```python
# Line 466
class SemanticClassifyExpr(ValidatedSignature, SemanticExpr):
    function_name = "semantic.classify"

    def __init__(
        self,
        expr: LogicalExpr,
        classes: List[ResolvedClassDefinition],
        temperature: float,
        examples: Optional[ClassifyExampleCollection] = None,
        model_alias: Optional[ResolvedModelAlias] = None,
    ):
        self.expr = expr          # The input column (e.g., "content")
        self.classes = classes    # The user-defined list of ClassDefinitions
        
        # ... validation logic ensures 'classes' are valid before proceeding ...
```

---

## 2. The Compiler Layer: Bridging Fenic to Polars

The bridge between Fenic's logical plan and the actual data processing engine is the **Compiler**. Fenic uses **Polars** as its physical execution engine due to its high performance and expressive API.

When an action (like `.show()` or `.collect()`) triggers execution, the `ExprConverter` class traverses the Fenic Logical Plan and converts every node into a native Polars expression.

### The `map_batches` Bridge
The key mechanism Fenic uses to inject AI capabilities into Polars is `map_batches`. This function allows developers to execute custom Python code on chunks (Series) of data within the Polars pipeline.

To Polars, the massive LLM operation is simply a user-defined function (UDF) that transforms a Series of Strings into another Series of Strings.

**File:** `src/fenic/_backends/local/transpiler/expr_converter.py`

```python
# Line 560
    @_convert_expr.register(SemanticClassifyExpr)
    def _convert_semantic_classify_expr(self, logical: SemanticClassifyExpr) -> pl.Expr:
        # 1. Define a closure that encapsulates the AI runtime logic.
        #    This function will be called by Polars with a chunk of data.
        def sem_classify_fn(batch: pl.Series) -> pl.Series:
            return SemanticClassify(
                input=batch,          # The raw text data from Polars
                classes=logical.classes,
                model=self.session_state.get_language_model(logical.model_alias),
                # ... other configuration parameters ...
            ).execute()

        # 2. Convert the input expression (e.g., col("content")) to a Polars expression.
        # 3. Attach the AI logic using .map_batches().
        #    'return_dtype=pl.Utf8' tells Polars to expect Strings in return.
        return self._convert_expr(logical.expr).map_batches(
            sem_classify_fn, return_dtype=pl.Utf8
        )
```

---

## 3. The Runtime Operator: The AI Engine

Inside the `map_batches` function, the `SemanticClassify` (aliased from `Classify`) class takes over. This class is the "engine room" responsible for the actual work of semantic processing. It treats the Polars Series as a list of inputs to be processed.

Its primary responsibilities are:
1.  **System Prompt Construction:** dynamically building the instructions for the LLM.
2.  **Request Management:** Handling the interaction with the `LanguageModel` interface.
3.  **Post-Processing:** validating and parsing the LLM's response.

**File:** `src/fenic/_backends/local/semantic_operators/classify.py`

```python
# Line 31
class Classify(BaseSingleColumnInputOperator[str, str]):
    SYSTEM_PROMPT = (
        "You are a text classification expert. "
        "Classify the following document into one of the following labels:"
        "\n{classes}\n"
        "Respond with *only* the predicted label."
    )

    # ...
    
    def build_system_message(self) -> str:
        # Dynamically injects the user's class labels and descriptions into the prompt
        class_descriptions = []
        for class_def in self.classes:
            # ... formatting logic ...
        
        return self.SYSTEM_PROMPT.format(classes=classes_text)
```

---

## 4. Structured Output Enforcement

One of Fenic's core value propositions is reliability. It does not rely on the LLM "promising" to follow instructions. Instead, it uses **Structured Output Enforcement** to mathematically constrain the LLM's response.

This is achieved by dynamically generating a strict schema contract (using Pydantic and Python Enums) that represents the user's allowed labels.

### Dynamic Model Generation
When the `Classify` operator is initialized, it calls a utility function to create a Pydantic model on the fly. This model restricts the `output` field to be exactly one of the user's provided labels.

**File:** `src/fenic/_backends/local/semantic_operators/utils.py`

```python
# Line 30
def create_classification_pydantic_model(allowed_values: List[str]) -> type[BaseModel]:
    # 1. Convert the list of allowed strings into a strict Python Enum.
    #    Example: ["urgent", "routine"] -> Enum { URGENT="urgent", ROUTINE="routine" }
    enum_name = "LabelEnum"
    enum_members = {value.upper(): value for value in allowed_values}
    enum_cls = Enum(enum_name, enum_members)

    # 2. Create a Pydantic Model where the 'output' field MUST be one of these Enum values.
    #    This effectively generates a JSON Schema: { "output": "urgent" | "routine" }
    return create_model(
        "EnumModel",
        output=(enum_cls, ...), # "..." indicates the field is required
    )
```

### Schema Enforcement
This dynamic Pydantic model is then passed to the inference configuration. Fenic's inference layer (and the underlying model providers like OpenAI or Gemini) uses this model to generate a JSON Schema.

During generation, the LLM is constrained (via token masking or constrained decoding) to *only* produce tokens that result in a valid JSON object matching this schema. This guarantees that the output will always be one of the defined labels, eliminating hallucinated categories.

**File:** `src/fenic/_backends/local/semantic_operators/classify.py`

```python
# Line 55
        self.output_model = create_classification_pydantic_model(labels)
        super().__init__(
            input,
            CompletionOnlyRequestSender(
                # ...
                inference_config=InferenceConfiguration(
                    # ...
                    # The dynamic model is passed here to enforce structure
                    response_format=ResolvedResponseFormat.from_pydantic_model(
                        self.output_model, generate_struct_type=False
                    ),
                ),
            ),
            examples,
        )
```