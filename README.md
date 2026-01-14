# SQL Agent - Hackapizza

SQL-Agent-Hackapizza is a project developed during IBM & Datapizza's Hackathon in January 2025, an event focused on Generative AI. The project aims to create an AI agent capable of interacting with SQL databases to manage and optimize recipes in a multi-dimensional culinary universe.

This project utilizes **LangGraph** for orchestrating agents and **Vanna AI** for generating SQL queries through LLMs. The core AI model used is **Mistral Large**, accessed via IBM's **WatsonX** platform.

![Graph](https://raw.githubusercontent.com/grct/SQL-Agent-Hackapizza/refs/heads/main/docs/chart.png)

## Table of Contents

- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Pipeline Architecture](#pipeline-architecture)
- [Database Schema](#database-schema)
- [Query Solving Workflow](#query-solving-workflow)
- [Setup & Configuration](#setup--configuration)

## Technology Stack

| Technology | Purpose |
|------------|---------|
| **LangGraph** | Agent orchestration and state management |
| **Vanna AI** | Text-to-SQL query generation |
| **Mistral Large** | Core LLM for natural language understanding |
| **IBM WatsonX** | LLM hosting platform |
| **OpenAI GPT-4o-mini** | SQL query generation via Vanna |
| **ChromaDB** | Vector store for Vanna's training data |
| **MySQL** | Database for storing culinary data |
| **LangChain** | LLM integration and prompt management |

## Project Structure

```
SQL-Agent-Hackapizza/
├── src/                          # Main source code
│   ├── main.py                   # Entry point - builds and runs the LangGraph pipeline
│   ├── models.py                 # State definitions for the graph
│   ├── settings.py               # LLM and database configuration
│   ├── training.py               # Vanna AI training with domain knowledge
│   ├── semaphore.py              # Thread synchronization for Vanna calls
│   ├── qa.py                     # Question-answering batch processing
│   ├── uploader.py               # Results CSV formatting utility
│   ├── nodes/                    # LangGraph node implementations
│   │   ├── merger.py             # Combines tool results into final answer
│   │   ├── splitter.py           # Decomposes complex queries
│   │   └── evaluator.py          # Evaluates response quality
│   └── tools/                    # LangChain tools for database queries
│       ├── ingredienti.py        # Search dishes by ingredient
│       ├── tecniche.py           # Search dishes by cooking technique
│       ├── licenze.py            # Search dishes by chef licenses
│       ├── sostanze.py           # Search dishes by nutritional substances
│       └── distanza.py           # Search dishes by restaurant distance
├── db/                           # Database setup scripts
│   ├── Script_Creazione_Tabelle.sql  # Table creation SQL
│   ├── Distanze.csv              # Planet distance matrix data
│   ├── popola_distance_matrix.py # Distance data population script
│   ├── popola_licenze.py         # License data population script
│   └── popola_tecniche.py        # Techniques data population script
├── preprocessing/                # Data extraction and loading scripts
│   ├── extractMenu.py            # Extract dishes from PDF menus
│   ├── extractTecniche.py        # Extract cooking techniques from PDFs
│   └── repository.py             # Database population from JSON files
├── docs/                         # Documentation and training data
│   ├── vanna/                    # Vanna AI training documentation
│   │   ├── relazioni.txt         # Database relationships documentation
│   │   ├── piatti.txt            # Dishes table documentation
│   │   ├── ingredienti.txt       # Ingredients table documentation
│   │   ├── tecniche.txt          # Techniques table documentation
│   │   ├── licenze.txt           # Licenses table documentation
│   │   ├── ristorante.txt        # Restaurant table documentation
│   │   └── ...                   # Other table documentation
│   ├── menu/                     # PDF menu files
│   ├── json/                     # Extracted JSON data
│   └── chart.png                 # Pipeline architecture diagram
└── pyproject.toml                # Python dependencies and project config
```

## Pipeline Architecture

The system implements a multi-agent architecture using LangGraph to process complex culinary queries. The pipeline consists of three main stages:

### 1. Router Node

The **Router** node receives user queries and uses the LLM (Mistral Large) with bound tools to determine which specialized tools should be invoked. It analyzes the natural language query to identify:

- Ingredients mentioned
- Cooking techniques required
- Chef licenses/certifications needed
- Substance/nutritional constraints
- Distance-based restaurant filters

### 2. Tools Node

The **Tools** node executes the selected tools in parallel. Each tool uses **Vanna AI** to translate specific sub-queries into SQL:

| Tool | Description | Example Query |
|------|-------------|---------------|
| `tool_ingredienti` | Find dishes containing a specific ingredient | "Dishes with Chocobo Wings" |
| `tool_tecniche` | Find dishes using a specific cooking technique | "Dishes using Gravitational Infusion Marination" |
| `tool_licenze` | Find dishes from chefs with specific certifications | "Dishes from Level 3 certified chefs" |
| `tool_sostanza` | Find dishes based on nutritional substance limits | "Dishes with CRP > 0.90" |
| `tool_distanza` | Find dishes from restaurants within distance constraints | "Dishes within 126 light-years from Cybertron" |

### 3. Merger Node

The **Merger** node combines results from all executed tools and applies the LLM to:

1. Intersect/filter results based on the original query logic
2. Extract the relevant dish IDs
3. Return the final answer as a JSON array of dish IDs

### Pipeline Flow

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌────────┐
│  START  │────▶│  Router │────▶│  Tools  │────▶│ Merger │────▶ END
└─────────┘     └─────────┘     └─────────┘     └────────┘
                    │               │
                    │    ┌──────────┴──────────┐
                    │    │   Parallel Tool     │
                    │    │    Execution        │
                    │    │                     │
                    │    ▼                     ▼
                    │ ┌────────┐         ┌────────┐
                    │ │Ingredi-│         │Tecniche│
                    │ │  enti  │         │        │
                    │ └────────┘         └────────┘
                    │ ┌────────┐         ┌────────┐
                    └▶│Licenze │         │Sostanze│
                      └────────┘         └────────┘
```

## Database Schema

The database models a multi-dimensional culinary universe with restaurants across different planets:

### Entity-Relationship Diagram

```
┌──────────────┐       ┌───────────────────┐       ┌──────────────┐
│  RISTORANTE  │◀─────▶│ RISTORANTE_PIATTI │◀─────▶│    PIATTI    │
│              │       └───────────────────┘       │              │
│ - id         │                                   │ - id         │
│ - nome       │       ┌───────────────────┐       │ - nome       │
│ - pianeta    │◀─────▶│RISTORANTE_LICENZE │       │              │
│ - chef       │       └───────────────────┘       └──────┬───────┘
└──────────────┘               │                          │
                               ▼                          │
                        ┌──────────────┐                  │
                        │   LICENZE    │     ┌────────────┼────────────┐
                        │              │     │            │            │
                        │ - id         │     ▼            ▼            ▼
                        │ - nome       │ ┌────────┐  ┌────────┐  ┌────────┐
                        │ - sigla      │ │PIATTI_ │  │PIATTI_ │  │PIATTI_ │
                        │ - livello    │ │INGREDI-│  │TECNICHE│  │SOSTANZE│
                        │ - descrizione│ │ENTI    │  │        │  │        │
                        └──────────────┘ └────┬───┘  └────┬───┘  └────────┘
                                              │           │
                                              ▼           ▼
                                       ┌──────────┐ ┌──────────┐
                                       │INGREDIEN-│ │ TECNICHE │
                                       │TI        │ │          │
                                       │          │ │ - id     │
                                       │ - id     │ │ - tipo   │
                                       │ - nome   │ │ - vantaggi│
                                       └──────────┘ │-svantaggi│
                                                    │-descrizione│
                                                    └──────────┘
```

### Tables Description

| Table | Description |
|-------|-------------|
| `PIATTI` | Dishes with unique ID and name |
| `INGREDIENTI` | Ingredients used in dishes |
| `TECNICHE` | Cooking techniques with advantages/disadvantages |
| `LICENZE` | Chef certifications with levels |
| `RISTORANTE` | Restaurants with planet location and chef |
| `DISTANZE` | Distance matrix between planets (in light-years) |
| `PIATTI_INGREDIENTI` | Many-to-many: dishes ↔ ingredients |
| `PIATTI_TECNICHE` | Many-to-many: dishes ↔ techniques |
| `PIATTI_SOSTANZE` | Nutritional substances per dish |
| `RISTORANTE_LICENZE` | Many-to-many: restaurants ↔ licenses |
| `RISTORANTE_PIATTI` | Many-to-many: restaurants ↔ dishes |

## Query Solving Workflow

### Step 1: Query Reception

A user submits a complex natural language query, for example:

> "Which dishes can I eat that contain Fusilli del Vento but have in their preparation the Gravitational Infusion Marination correctly operated by a chef who has the correct licenses and certifications described by the Galactic Code?"

### Step 2: Tool Selection (Router)

The Router analyzes the query and identifies that it requires:
- `tool_ingredienti` for "Fusilli del Vento"
- `tool_tecniche` for "Marinatura a Infusione Gravitazionale"
- `tool_licenze` for "Codice Galattico" certifications

### Step 3: SQL Generation (Tools via Vanna)

Each tool uses Vanna AI to generate SQL queries:

```sql
-- tool_ingredienti: Find dishes with "Fusilli del Vento"
SELECT id_piatto FROM PIATTI_INGREDIENTI PI
INNER JOIN INGREDIENTI I ON PI.id_ingrediente = I.id
WHERE I.nome LIKE '%Fusilli del Vento%'
-- Returns: [12, 45, 78, 112, 156]

-- tool_tecniche: Find dishes using "Marinatura a Infusione Gravitazionale"
SELECT PT.id_piatto FROM PIATTI_TECNICHE PT
INNER JOIN TECNICHE T ON PT.id_tecnica = T.id
WHERE T.descrizione LIKE '%Marinatura a Infusione Gravitazionale%'
-- Returns: [32, 45, 67, 78, 89, 112]

-- tool_licenze: Find dishes from restaurants with Galactic Code certifications
SELECT RP.id_piatto FROM RISTORANTE_PIATTI RP
INNER JOIN RISTORANTE R ON RP.id_ristorante = R.id
INNER JOIN RISTORANTE_LICENZE RL ON R.id = RL.id_ristorante
INNER JOIN LICENZE L ON RL.id_licenza = L.id
WHERE L.nome LIKE '%Codice Galattico%'
-- Returns: [45, 78, 112, 134, 167]
```

### Step 4: Result Aggregation (Merger)

The Merger combines results from all tools:
1. Collects all dish IDs from each tool execution
2. Applies intersection logic based on AND conditions in the original query
3. Filters to only include dishes meeting ALL criteria

```
Ingredienti results: [12, 45, 78, 112, 156]
Tecniche results:    [32, 45, 67, 78, 89, 112]
Licenze results:     [45, 78, 112, 134, 167]
                     ────────────────────────
Intersection:        [45, 78, 112]  ← Final answer
```

### Step 5: Final Response

Returns a JSON array of matching dish IDs:

```json
[45, 78, 112]
```

## Setup & Configuration

### Environment Variables

```bash
# IBM WatsonX
PROJECT_ID=your_watsonx_project_id

# OpenAI (for Vanna)
OPENAI_API_KEY=your_openai_api_key

# MySQL Database
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=HackaPizza
```

### Installation

```bash
# Install dependencies using Poetry
poetry install

# Or using pip
pip install -r requirements.txt
```

### Running the Agent

```bash
# Run the main pipeline
python -m src.main

# Process batch questions
python -m src.qa
```

## Contributing

This project was developed during the IBM & Datapizza Hackathon. Contributions are welcome via pull requests.

## License

This project is part of the Hackathon challenge and follows the event's licensing terms.
