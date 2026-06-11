# GRAPHON — Knowledge Graph QA System
## Architecture & Engineering Specification
**Classification:** JPMC Internal | Intelligence Tooling Suite | v1.0

---

## Architecture Pattern

> **Schema-Grounded KGQA × Hybrid NL2Cypher Pipeline × GraphRAG Faithful Synthesis**

---

## 1. Canonical Names

| Term | Definition |
|---|---|
| **KGQA** (Knowledge Graph Question Answering) | The problem. User asks English questions; system answers from a knowledge graph. |
| **GraphRAG** (Graph Retrieval-Augmented Generation) | The paradigm. LLM answers are grounded only in retrieved subgraphs — not parametric memory. |
| **NL2Cypher / Text-to-Cypher** | The query generation step. Natural language → valid Cypher, constrained by live schema. |
| **Schema-Grounded Generation** | Anti-hallucination constraint. LLM forbidden from inventing labels/types not in registry. |
| **Grounded / Faithful Generation** | Answer synthesis contract. Every claim traced to a specific node or relationship ID. |

**One-line pitch:**
*"GRAPHON is a schema-grounded KGQA system with a hybrid NL2Cypher pipeline and GraphRAG-style faithful answer synthesis over the JPMC relationship graph."*

---

## 2. Pipeline — 6 Layers

```
┌─────────────────────────────────────────────┐
│           USER QUESTION (NLQ)               │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│  LAYER 1 — Intent & Entity Extraction       │
│  ┌─────────────────┐ ┌───────────────────┐  │
│  │ Intent          │ │ Entity            │  │
│  │ Classifier      │ │ Extractor         │  │
│  │ Rule-based +    │ │ NER + fuzzy       │  │
│  │ LLM fallback    │ │ match vs live DB  │  │
│  └─────────────────┘ └───────────────────┘  │
│  ┌─────────────────┐                         │
│  │ Ambiguity       │                         │
│  │ Resolver        │                         │
│  │ Clarify / best  │                         │
│  └─────────────────┘                         │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│  LAYER 2 — Schema & Context Grounding       │
│  ⚠️  ANTI-HALLUCINATION GATE #1              │
│  ┌─────────────────┐ ┌───────────────────┐  │
│  │ Schema          │ │ Context           │  │
│  │ Registry        │ │ Injector          │  │
│  │ Live Neo4j      │ │ Schema → prompt   │  │
│  │ introspection   │ │ at runtime        │  │
│  └─────────────────┘ └───────────────────┘  │
│  ┌─────────────────┐                         │
│  │ Query           │                         │
│  │ Planner         │                         │
│  │ single/multi-   │                         │
│  │ hop decision    │                         │
│  └─────────────────┘                         │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│  LAYER 3 — Query Generation (NL2Cypher)     │
│  ┌─────────────────┐ ┌───────────────────┐  │
│  │ Template        │ │ LLM Cypher        │  │
│  │ Matcher         │ │ Generator         │  │
│  │ Deterministic   │ │ Schema-fenced     │  │
│  │ fast path       │ │ few-shot prompt   │  │
│  └─────────────────┘ └───────────────────┘  │
│  ┌─────────────────┐                         │
│  │ Self-Correction │◄──── retry ≤3×          │
│  │ Loop            │                         │
│  │ Syntax check    │                         │
│  └─────────────────┘                         │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│  LAYER 4 — Execution & Validation           │
│  ⚠️  ANTI-HALLUCINATION GATE #2              │
│  ┌─────────────────┐ ┌───────────────────┐  │
│  │ Graph DB        │ │ Result            │  │
│  │ Executor        │ │ Validator         │  │
│  │ Neo4j read-only │ │ Empty/type check  │  │
│  │ transactions    │ │ honest "no data"  │  │
│  └─────────────────┘ └───────────────────┘  │
│  ┌─────────────────┐                         │
│  │ Result Cache    │                         │
│  │ TTL-keyed       │                         │
│  │ Redis / memory  │                         │
│  └─────────────────┘                         │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│  LAYER 5 — Answer Synthesis (Grounded Gen)  │
│  ┌─────────────────┐ ┌───────────────────┐  │
│  │ Graph Data      │ │ NL Answer         │  │
│  │ Serialiser      │ │ Generator         │  │
│  │ nodes/edges     │ │ Data-fenced       │  │
│  │ → JSON payload  │ │ prompt only       │  │
│  └─────────────────┘ └───────────────────┘  │
│  ┌─────────────────┐                         │
│  │ Citation        │                         │
│  │ Attacher        │                         │
│  │ claim → nodeID  │                         │
│  └─────────────────┘                         │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│  LAYER 6 — Visual Rendering                 │
│  ┌─────────────────┐ ┌───────────────────┐  │
│  │ Layout Engine   │ │ Graph Renderer    │  │
│  │ dagre/force/    │ │ ReactFlow +       │  │
│  │ radial/recharts │ │ custom nodes      │  │
│  └─────────────────┘ └───────────────────┘  │
│  ┌─────────────────┐                         │
│  │ Answer Panel    │                         │
│  │ NL text + graph │                         │
│  │ citation-linked │                         │
│  └─────────────────┘                         │
└─────────────────────────────────────────────┘
```

---

## 3. Anti-Hallucination Contract

### Rule 1 — Schema Fence
- The LLM cannot reference vocabulary not in the injected schema.
- Post-generation Cypher AST validator rejects unknown node labels or relationship types **before execution**.
- Failure → self-correction retry. Never silent execution of hallucinated labels.

### Rule 2 — Answer Grounding
- System prompt: *"You are a report writer. The only facts you may state are in the data below. If the data does not contain the answer, say so explicitly."*
- No general knowledge. No inference beyond returned records.
- Every factual sentence must cite a node ID or relationship ID.

---

## 4. LLM Prompt Templates

### 4.1 Layer 3 — NL2Cypher System Prompt
```
You are a Cypher query generator for a Neo4j knowledge graph.
You must only use the node labels, relationship types, and property names
listed in the SCHEMA section below. Never invent new labels or types.
Respond with a single valid Cypher query and nothing else.
Do not include explanations, markdown, or backticks.

SCHEMA:
  Node labels         : {schema.node_labels}
  Relationship types  : {schema.rel_types}
  Properties by label : {schema.properties_json}

EXAMPLES:
  Q: Who does Alice work for?
  A: MATCH (p:Person {name: 'Alice'})-[:WORKS_AT]->(c:Company) RETURN c.name

  Q: Find all products supplied by companies in Mumbai.
  A: MATCH (city:City {name:'Mumbai'})<-[:LOCATED_IN]-(co:Company)
     -[:SUPPLIES]->(pr:Product) RETURN co.name, pr.name
```

### 4.2 Layer 3 — User Prompt
```
Translate the following question into a Cypher query:
{user_question}

Detected intent  : {intent}
Resolved entities: {entities_json}
```

### 4.3 Layer 3 — Self-Correction Re-prompt
```
Your previous query contained an error:
{cypher_error_message}

Original query:
{previous_cypher}

Fix the Cypher. Use only the schema vocabulary above.
Return only the corrected query.
```

### 4.4 Layer 5 — Faithful Answer Generation System Prompt
```
You are a data analyst writing a factual summary for a business stakeholder.
The ONLY facts you may state are those present in the DATA section below.
If the data does not contain enough information to answer the question, say:
"The available records do not contain sufficient information to answer this question."
Do not use general knowledge. Do not infer relationships not present in the data.
For every factual claim you make, append a citation in the format [node:<id>]
or [rel:<id>] referencing the node or relationship ID from the data.

DATA:
{serialised_graph_json}
```

### 4.5 Layer 5 — User Prompt
```
Question: {user_question}
Write a clear, concise answer (3-5 sentences) using only the data above.
```

---

## 5. Technology Stack

| Component | Role | Category |
|---|---|---|
| FastAPI (Python) | Pipeline orchestrator, schema introspection, LLM proxy, cache | Backend |
| Neo4j 5.x + APOC | Primary graph DB, schema introspection source of truth | Graph DB |
| Claude API (claude-sonnet-4) | Intent fallback, Cypher generation, answer synthesis | LLM |
| React 18 + TypeScript | Frontend shell, dual-panel UI, citation cross-highlighting | Frontend |
| ReactFlow | Graph renderer with custom node/edge components | Visualisation |
| Recharts | Aggregation charts for count/sum/avg query results | Visualisation |
| spaCy | NER for entity extraction (Layer 1) | NLP |
| Redis | TTL-keyed result cache (Layer 4) | Cache |
| Pydantic v2 | Schema registry models, request/response validation | Validation |
| LangGraph (optional) | Multi-step agentic query planning for complex queries | Orchestration |

---

## 6. Internal Naming Conventions

| Concept | GRAPHON Canonical Name |
|---|---|
| The system | GRAPHON |
| The architecture pattern | Schema-Grounded KGQA |
| The retrieval paradigm | GraphRAG |
| The query translation step | Hybrid NL2Cypher pipeline |
| Hallucination prevention (L2) | Schema Fence |
| Hallucination prevention (L5) | Answer Grounding |
| Module: schema introspection | SchemaRegistry |
| Module: entity resolution | EntityResolver |
| Module: query routing | QueryPlanner |
| Module: template-based query | TemplateEngine |
| Module: LLM query generation | CypherGenerator |
| Module: syntax validation | CypherValidator |
| Module: DB execution | GraphExecutor |
| Module: answer synthesis | AnswerSynthesiser |
| Module: graph serialisation | GraphSerialiser |
| Module: citation attachment | CitationAttacher |
| Module: layout selection | LayoutSelector |
| Module: graph render | GraphRenderer |

---

## 7. Hallucination Prevention Checklist

- [ ] SchemaRegistry populated from live DB introspection — not a static file
- [ ] CypherValidator rejects unknown labels before GraphExecutor is called
- [ ] AnswerSynthesiser system prompt contains exact data-fencing instruction (Section 4.4)
- [ ] Empty DB results return honest "no data found" — LLM never receives an empty payload
- [ ] CitationAttacher links every factual sentence to a node ID or relationship ID
- [ ] Dual-panel UI allows claim verification by clicking citation → source subgraph highlight
- [ ] All LLM calls logged with full prompt, injected schema, and raw response
- [ ] TemplateEngine covers top-10 query intents — LLM never called for these
- [ ] Self-correction loop capped at 3 retries
- [ ] GraphExecutor uses read-only Neo4j transactions — NLQ pipeline cannot mutate the graph

---

*GRAPHON | JPMC Intelligence Tooling Suite | Internal*
