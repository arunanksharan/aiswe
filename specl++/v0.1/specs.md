Understood. You require a more concrete, exhaustive definition of the keys and potential values within the `SpecL++` v0.2 framework, suitable as a starting point for implementation, rather than just the abstract structure or a single example.

Let's break down each top-level key and its expected structure or value set, focusing on the scope required for Phase 1 (simple CRUD APIs). This is presented based on first principles for clarity and automation.

**`SpecL++` v0.2 Exhaustive Details (Phase 1 Scope):**

1.  **`specVersion`** (Mandatory)

    - **Type:** `string`
    - **Value:** Fixed `"0.2"` for this version.
    - **Purpose:** Ensures the parser (`Synapse++`) uses the correct syntax rules.

2.  **`goalDescription`** (Mandatory)

    - **Type:** `string` (potentially multi-line)
    - **Value:** Free-form text describing the business or user objective. Should be clear enough for human understanding and potential LLM context.
    - **Example:** `"Generate a backend API for internal inventory tracking, allowing authorized users to add, view, update, and remove stock items based on the Product ontology."`

3.  **`applicationType`** (Mandatory)

    - **Type:** `string` (Enum recommended in implementation)
    - **Purpose:** Selects the primary generation pattern for `Synapse++`.
    - **Phase 1 Value:**
      - `CRUD_API`: Generate a standard RESTful API (potentially GraphQL later) with endpoints for Create, Read, Update, Delete, List operations on the `targetEntity`. Requires the `endpoints` key.
    - **Future Potential Values:** `Web_Form_App`, `Batch_Process`, `Data_Pipeline`, `Event_Driven_Service`.

4.  **`targetEntity`** (Mandatory for `CRUD_API`)

    - **Type:** `string` (Ontology Reference)
    - **Format:** `ontology:<ClassName>`
    - **Purpose:** Specifies the primary Core Ontology Pydantic model (from Phase 0) that the CRUD operations revolve around.
    - **Example Values:** `ontology:Person`, `ontology:Product`, `ontology:Order`, `ontology:Event`, `ontology:Organization`. Must match a generated Pydantic class name derived from the ontology.

5.  **`globalConfig`** (Optional)

    - **Type:** `map` (`string` -> `CapabilityInstanceDefinition`)
    - **Purpose:** Define reusable capability instances or configurations. Reduces repetition in endpoint workflows. Keys are arbitrary unique names. Referenced via YAML anchors (`&name`/`*name`) or potentially dot notation (`globalConfig.name`).
    - **Structure (`CapabilityInstanceDefinition`):**
      - `capability`: `string` (`VC:<CapabilityName>`) - The type of VC being configured.
      - `config`: `map` (VC-specific configuration parameters using static values, `env:VAR`, or `llm:hint`).
      - `output`: `string` (Optional) - Assigns a default name to the output of this global instance if it's always the same (e.g., `output: standardClaims` for an AuthN provider).
    - **YAML Example:**
      ```yaml
      globalConfig:
        standardAuthN: &stdAuthN # Defines &stdAuthN anchor
          capability: VC:AuthN_JWTBearer
          config:
            jwks_url: 'env:AUTH_JWKS_URL'
            audience: 'urn:my-api'
            issuer: 'env:AUTH_ISSUER_URL'
          output: verifiedClaims
        mainDatabase: &mainDB
          capability: VC:BasicPersistenceAdapter[InMemory] # Or a specific SQL one later
          # config: { connection_string_env: "DB_CONN_STR" }
      ```

6.  **`projectionModels`** (Optional)

    - **Type:** `map` (`ProjectionName` -> `ProjectionDefinition`)
    - **Purpose:** Define named data subsets derived from Core Ontology models. Used for API request/response bodies or internal data flow.
    - **Structure (`ProjectionDefinition`):**
      - `basedOn`: `string` (`ontology:<ClassName>`) - Mandatory.
      - `description`: `string` - Optional.
      - `includeFields`: `list[string]` - Mandatory. List of snake_case field names present in the base Pydantic model.
    - **YAML Example:**
      ```yaml
      projectionModels:
        ProductSummary:
          basedOn: ontology:Product
          description: 'Minimal product info for listings.'
          includeFields: [id, name, sku, brand_name] # Assume Product has brand_name
        UserInput:
          basedOn: ontology:Person
          description: 'Fields needed to create/update a user.'
          includeFields: [givenName, familyName, email]
      ```

7.  **`endpoints`** (Mandatory for `CRUD_API`)

    - **Type:** `list` (of `EndpointDefinition` maps)
    - **Purpose:** Defines each API endpoint.
    - **Structure (`EndpointDefinition`):**
      - `path`: `string` (Mandatory, e.g., `/products`, `/products/{productId}`).
      - `method`: `string` (Mandatory, e.g., `GET`, `POST`, `PUT`, `DELETE`).
      - `summary`: `string` (Optional).
      - `parameters`: `list` (Optional, of `ParameterDefinition` maps)
        - `name`: `string` (Mandatory).
        - `in`: `path` | `query` (Mandatory).
        - `required`: `boolean` (Mandatory).
        - `schema`: `map` (Mandatory)
          - `typeRef`: `string` (`ontology:<TypeName>` or `string`, `integer`, `number`, `boolean`).
          - `description`: `string` (Optional).
      - `requestBody`: `map` (Optional)
        - `required`: `boolean`.
        - `projection`: `string` (`projection:<ProjectionName>`).
      - `responses`: `map` (Mandatory)
        - `'<HTTPStatusCode>'`: `map` (String keys like `'200'`, `'404'`)
          - `description`: `string` (Mandatory).
          - `projection`: `string` (Optional, `projection:<ProjectionName>`) - Defines response body structure.
      - `workflow`: `list` (Mandatory, of `WorkflowStepDefinition` maps).

8.  **`workflow` Step (`WorkflowStepDefinition`)**

    - **Type:** `map`
    - **Purpose:** Defines a single capability execution within a workflow.
    - **Structure:**
      - `stepId`: `string` (Mandatory, unique within the workflow).
      - `capability` OR `capabilityRef`: `string` (Mandatory, choose one).
        - `capability`: Specifies VC type directly (e.g., `VC:CRUDHandler`). Phase 1 values: `VC:RESTEndpoint`, `VC:CRUDHandler`, `VC:BasicPersistenceAdapter[InMemory]`, `VC:AuthN_JWTBearer`, `VC:AuthZ_SimpleRBAC`.
        - `capabilityRef`: References a globally defined capability instance (e.g., `globalConfig.standardAuthN`, `*stdAuthN`).
      - `config`: `map` (Optional). VC-specific parameters. Examples:
        - For `VC:AuthZ_SimpleRBAC`: `required_role: <string>`, `role_claim_path: <string>`
        - For `VC:CRUDHandler`: `entity: <ontology:ClassName>`, `persistenceAdapterRef: <ref>`, `operation: Create|Read|Update|Delete|List`
        - For `VC:RESTEndpoint`: `path`, `method`, `requestModel`, `responseModel`, `handlerSequence: [<stepId1>, <stepId2>]`
        - Can include `env:<VAR>` or `llm:<hint>` placeholders.
      - `input`: `string` (Optional). Reference to data source: `requestBody`, `pathParams.<name>`, `queryParams.<name>`, `<prevStepId>.<outputName>`.
      - `inputProjection`: `string` (Optional, `projection:<Name>`).
      - `output`: `string` (Optional, name for this step's result).
      - `outputProjection`: `string` (Optional, `projection:<Name>`).

9.  **`nfrs`** (Optional)
    - **Type:** `map` (`string` -> `any`)
    - **Purpose:** Specify non-functional requirements as hints.
    - **Example Keys for Phase 1:**
      - `target_response_time_ms`: `integer`
      - `securityLevel`: `"Low" | "Medium" | "High"`
      - `compliance`: `list[string]` (e.g., `["PII_Basic"]`)

This provides a more exhaustive definition of the `SpecL++` v0.2 structure and the expected values or formats for its keys, sufficient to start writing specifications for simple CRUD APIs and build the corresponding parser and synthesis logic.
