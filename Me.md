Repository Impact Analysis Agent — Graph Architecture Context

1. Project Overview

I am building a Repository Impact Analysis Agent that analyzes multiple potentially interdependent code repositories and identifies the files, functions, services, configurations, validations, and tests affected by a requested change.

Example use case

A user requests:

> Reduce the maximum allowed username length from 70 characters to 20 characters.



The agent should identify:

- All locations where username constraints are defined.
- Related names and aliases (username, userName, loginId, etc.), but only where relationships can be verified.
- Frontend and backend validations.
- Error messages referencing the old constraint.
- Shared schemas and API contracts.
- Functions and services depending on the relevant code.
- Downstream consumers across repositories.
- Associated test cases.
- Potential database constraints and existing-data implications.

Primary goal: Identify what needs to change, where, and why, without necessarily modifying code.

The system should be generic and work across different languages, frameworks, repositories, and types of change requests.


---

2. Core Architectural Decision

We decided to build a typed, directed Code Knowledge Graph, primarily stored in PostgreSQL.

Constraints and preferences:

- No Neo4j or Amazon Neptune assumed.
- PostgreSQL is the preferred persistent graph storage.
- NetworkX can optionally be used for in-memory traversal or prototyping.
- The graph is built before user queries arrive.
- The Investigator Agent queries the graph through tools.
- The graph must be versioned and updated as repositories change.
- The graph is one retrieval mechanism, not the entire impact-analysis system.

Two main components

A. Repository Indexing / Graph Building Service

Responsible for:

1. Accessing Git repositories.
2. Discovering files and languages.
3. Parsing code using AST/Tree-sitter.
4. Resolving symbols and references through LSP or language-specific analysis.
5. Extracting API contracts, schemas, and dependencies.
6. Converting extracted information into graph nodes and edges.
7. Persisting the graph in PostgreSQL.
8. Updating graph snapshots when code changes.

B. Investigator Agent

Responsible for:

1. Understanding natural-language change requests.
2. Finding relevant starting points through search.
3. Resolving discovered code into graph nodes.
4. Traversing code dependencies.
5. Requesting additional source evidence.
6. Identifying direct and indirect impacts.
7. Producing a structured, evidence-backed impact report.

These should be logically separate components, although they can initially live in the same FastAPI application with a separate indexing worker.


---

3. What the Graph Should Contain

The graph must represent more than:

Repository → Folder → File → Class → Function

That hierarchy is useful but insufficient.

We need relationships between actual source-code entities.

Structural nodes

- REPOSITORY
- FILE
- CLASS
- FUNCTION
- METHOD
- FIELD
- CONSTANT
- SCHEMA
- API_ENDPOINT
- TEST

Behavioral / semantic nodes

These can be added as extractors become more sophisticated:

- VALIDATION_RULE
- ERROR_MESSAGE
- CONFIGURATION
- BUSINESS_ENTITY
- EVENT

Graph edges

Relationship	Meaning

CONTAINS	Repository contains file
DEFINES	File defines function/class/field
CALLS	Function calls another function
IMPORTS	File imports a module
REFERENCES	Symbol references another symbol
EXPOSES	Service exposes an API
CONSUMES	Another service or client consumes an API
VALIDATES_FIELD	Validation rule applies to a field
ENFORCES_RULE	Code enforces a constraint
USES_CONFIG	Function uses a configuration
EMITS_ERROR	Code emits a particular error
TESTS	Test verifies a function or behavior
PUBLISHES_EVENT	Component publishes an event
HANDLES_EVENT	Component consumes an event


Edges must be typed and directed.

An important consideration is that a dependency relationship does not automatically prove an impact. The agent must inspect the relationship's meaning and direction.


---

4. How the Graph Is Constructed

Step 1: Repository scanning

Use Git and filesystem utilities to discover tracked source files.

Possible tools:

- git clone
- git fetch
- git ls-files
- git rev-parse
- Python pathlib

Record repository IDs, file paths, language, branch, and commit SHA.

Step 2: Code parsing

Use language-specific AST parsers or Tree-sitter to extract:

- Classes.
- Functions and methods.
- Fields.
- Constants.
- Imports.
- Syntactic references.
- Source locations.

Python's built-in ast module can work for Python-only prototypes. Tree-sitter is useful for multilingual repositories.

Step 3: Symbol resolution using LSP

LSP = Language Server Protocol.

It is the protocol behind code-navigation features commonly found in IDEs.

It can provide:

- Go to Definition.
- Find References.
- Document Symbols.
- Type Definition.
- Call Hierarchy, where supported.

Example:

# validators.py  
MAX_USERNAME_LENGTH = 70  
  
def validate_username(username):  
    return len(username) <= MAX_USERNAME_LENGTH

# user_service.py  
from validators import validate_username  
  
def create_user(user):  
    validate_username(user.username)

Using parsing and symbol resolution, we want graph relationships such as:

create_user → CALLS → validate_username

validate_username → REFERENCES → MAX_USERNAME_LENGTH

LSP is especially useful for differentiating symbols with the same name.

However, its capabilities and accuracy depend on the language server and project configuration. Dynamic code may remain difficult to resolve.

Important division:

- Tree-sitter identifies syntactic structure.
- LSP helps resolve symbol identities and usages.
- PostgreSQL stores the resulting relationships.
- The Investigator Agent traverses those relationships.

LSP does not require VS Code. A language server can run as a separate process communicating over JSON-RPC.

Step 4: Framework-specific extraction

Generic AST and LSP analysis will not capture all relevant relationships.

We may need specialized extractors for:

- FastAPI route definitions.
- Pydantic validation rules.
- React frontend components.
- TypeScript interfaces.
- Java annotations.
- OpenAPI specifications.
- SQL schemas and migrations.
- Event/message consumers.

All extractors should produce a normalized graph representation.

Step 5: Cross-repository linking

Imports alone cannot identify all dependencies across repositories.

For example:

frontend.userApi → CONSUMES → POST /users → IMPLEMENTED_BY → backend.create_user

These relationships may be derived from:

- OpenAPI contracts.
- HTTP clients and route definitions.
- Shared schemas.
- Service dependency metadata.
- Event/message contracts.

Cross-repository edges should include evidence and a confidence or verification status.


---

5. PostgreSQL Graph Storage

We chose an adjacency-list-based relational graph.

Primary tables

graph_nodes

Suggested fields:

- id
- snapshot_id
- node_key
- node_type
- repository
- file_path
- symbol_name
- start_line
- end_line
- properties (JSONB)

graph_edges

Suggested fields:

- id
- snapshot_id
- source_id
- target_id
- edge_type
- properties (JSONB)

graph_snapshots

Stores graph version, indexing status, creation time, and revision metadata.

Repository revision information should also be tracked, potentially through repositories and repository_revisions tables.

Each edge should ideally preserve:

- Source and target identifiers.
- Edge type.
- Repository revision.
- Source location or supporting evidence.
- Extraction method.
- Verification/confidence status.

Use PostgreSQL indexes on source IDs, target IDs, snapshot IDs, and relevant node types.

For multi-hop traversal, use recursive SQL queries with WITH RECURSIVE.

Graph traversal should be bounded by depth, visited nodes, and returned result count.

NetworkX is optional, rather than mandatory.


---

6. How the Investigator Agent Uses the Graph

The Investigator Agent should never receive the complete graph in its LLM context.

Instead, expose deterministic graph-query tools.

Proposed graph tools

Tool	Purpose

find_nodes	Find nodes matching a symbol, file, repository, or type
get_neighbors	Retrieve directly connected nodes
traverse_graph	Explore dependencies over multiple hops
get_node_evidence	Retrieve source evidence for relationships
find_paths	Optionally find paths between two known graph nodes


Example:

{  
  "tool": "get_neighbors",  
  "arguments": {  
    "node_id": 3,  
    "direction": "incoming",  
    "edge_types": ["REFERENCES"]  
  }  
}

For the username example:

1. Exact search discovers MAX_USERNAME_LENGTH.
2. The agent identifies the corresponding graph node.
3. It retrieves incoming REFERENCES edges.
4. It discovers validate_username.
5. It follows incoming CALLS edges to identify callers.
6. It explores APIs, downstream consumers, and tests.
7. It reads actual source code to verify potential impacts.
8. It performs further searches when graph relationships are missing.
9. It produces an evidence-backed report.

Important distinction

The graph stores stable code facts, not predictions about future changes.

We should not precompute query-specific edges such as:

UserForm → WILL_BE_IMPACTED_BY → UsernameLengthChange

Instead, the investigator derives impacts dynamically based on the user's request and verified repository relationships.


---

7. Graph vs Exact Search vs Semantic Search

The complete agent needs all three.

Mechanism	Purpose	Example

Exact search	Locate literal names, values, and patterns	Find MAX_USERNAME_LENGTH, 70
Semantic search	Discover conceptually related code	Find login identifier validation under another name
Graph traversal	Explore connected dependencies	Find callers of a validation function


Expected investigation workflow:

User Request  
     ↓  
Exact / Semantic Search  
     ↓  
Candidate Files and Symbols  
     ↓  
Resolve to Graph Nodes  
     ↓  
Graph Traversal  
     ↓  
Related Functions / APIs / Tests  
     ↓  
Read Code and Verify  
     ↓  
More Search if Needed  
     ↓  
Impact Report

The graph cannot replace searching source text.

Search cannot replace dependency tracing.

Semantic similarity is not evidence that two fields represent the same business concept.

For example, lastName must not automatically be treated as an alias of username. The relationship requires supporting evidence from mappings, schemas, assignments, or actual data flow.


---

8. How We Keep the Graph Generic

The graph builder must be independent of the type of change requested.

We should not hardcode:

if field_name == "username":  
    find_username_validations()

Instead, build a modular indexing system:

Repository Scanner  
       ↓  
Language Detection  
       ↓  
AST / Tree-sitter Extractors  
       ↓  
LSP / Symbol Resolution  
       ↓  
Framework-specific Extractors  
       ↓  
Normalized Nodes and Edges  
       ↓  
PostgreSQL

Each extractor implements a common output contract for nodes, edges, and source evidence.

This allows support for new programming languages and frameworks without redesigning the overall graph.

Generic does not mean complete.

Unsupported languages, dynamic behavior, and framework-specific patterns must be tracked as indexing limitations.


---

9. Handling Code Updates and New Releases

The graph must be synchronized with Git.

Incremental indexing

When code is pushed:

1. Git webhook or CI event triggers indexing.
2. The worker identifies the previous and new commit SHAs.
3. git diff --name-status OLD_SHA NEW_SHA identifies changed files.
4. Changed files are reparsed.
5. Affected graph nodes and relationships are updated.
6. References from unchanged files are re-resolved when necessary.
7. A new graph snapshot is validated.
8. The completed snapshot is published atomically.

A function rename or API contract change may require a wider dependency refresh even when consuming files were not modified.

Versioned graphs

Each graph snapshot should be tied to exact repository commits.

Example:

Graph snapshot	Frontend revision	Backend revision

V1	a123	b456
V2	a789	b456
V3	a789	b999


The investigator should use a consistent, pinned snapshot for each request.

Do not combine evidence from different source revisions unintentionally.

For multiple repositories, maintain a revision manifest identifying the exact commit of each repository.

A graph snapshot establishes the code versions examined, but it does not automatically prove those versions were deployed together.

Deployment and recovery

- Build new graph snapshots without modifying the active version.
- Validate before publication.
- Atomically activate completed snapshots.
- Retain previous snapshots for rollback.
- Allow investigation against specific releases where needed.
- Report stale or partially indexed repository information.

For a small MVP, full reindexing is acceptable before optimizing for incremental updates.


---

10. How Graph Tools Fit Into the Overall Deterministic Toolset

The graph is one part of the complete agent.

Other relevant investigator tools include:

Tool	Role

list_repositories	Discover available repositories
list_files	Navigate source structure
search_exact	Search source text using ripgrep
read_file	Read targeted file ranges
find_symbols	Discover functions/classes/constants
go_to_definition	Resolve symbol declarations
find_references	Locate symbol usages
find_callers	Identify function callers
traverse_graph	Explore dependencies
inspect_schema	Inspect API and field constraints
find_validations	Discover validation rules
find_error_messages	Locate relevant error messages
find_tests	Locate relevant tests
inspect_config	Inspect configuration dependencies
inspect_database	Inspect database schemas and migrations
find_api_consumers	Locate cross-service API consumers


The LLM dynamically selects tools based on investigation progress.

The graph is especially valuable after a relevant symbol, API, or component has been discovered and the agent needs to explore its dependencies.

Agent-facing tools should be read-only, access-controlled, bounded, and return structured results.


---

11. Recommended Technology Stack

Component	Technology

Application	FastAPI
Agent orchestration	LangGraph or existing LangChain-based framework
Git synchronization	Git CLI / webhooks
Exact search	ripgrep
Code parsing	Tree-sitter, Python AST where appropriate
Symbol resolution	LSP
Graph persistence	PostgreSQL
Graph traversal	SQL recursive CTEs
Optional in-memory graph	NetworkX
Semantic retrieval	PostgreSQL + pgvector
Graph API	Python service functions exposed as agent tools


No Neo4j or Neptune required.


---

12. Main Design Principles

1. Build the graph before user queries, not on every investigation.
2. Separate repository indexing from agentic investigation.
3. Use typed, directed relationships between code entities.
4. Store source evidence for every meaningful relationship.
5. Use Tree-sitter for structure and LSP for symbol resolution.
6. Treat cross-repository API dependencies as first-class relationships.
7. Use PostgreSQL as the durable source of truth.
8. Expose narrow graph-query tools rather than the entire graph.
9. Combine exact search, semantic search, and graph traversal.
10. Do not equate a dependency with a confirmed change impact.
11. Keep graph snapshots tied to exact repository revisions.
12. Track unsupported extraction cases and indexing gaps explicitly.

13. Current Status and Next Steps

We have discussed the architecture conceptually but have not implemented or validated a graph builder against real repositories yet.

The next engineering tasks are:

1. Finalize the minimum node and edge schema.
2. Define a common extractor output contract.
3. Select the initial supported languages/frameworks.
4. Implement repository scanning and AST-based extraction.
5. Add LSP-based symbol and reference resolution.
6. Persist graph nodes and edges in PostgreSQL.
7. Implement find_nodes, get_neighbors, and traverse_graph.
8. Connect graph tools to the Investigator Agent.
9. Test an end-to-end example covering two interdependent repositories.
10. Measure whether traversal identifies genuine impacts without excessive false positives.

What I want to explore in the next conversation

Focus specifically on implementing the graph-building and graph-query capabilities, rather than restarting the high-level architecture discussion.

I want to understand:

- The exact graph schema needed for a realistic codebase.
- How to extract nodes and edges through Tree-sitter and LSP.
- How to reliably connect relationships across repositories.
- Which relationships can be extracted deterministically versus inferred.
- How to persist and traverse the graph efficiently in PostgreSQL.
- How to connect graph queries with the Investigator Agent.
- How to test graph accuracy, coverage, and update correctness.

The goal is to move from a conceptual graph architecture to a practical, generic, testable implementation.
