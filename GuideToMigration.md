# GuideToMigration.md — NaturalSQL: Migración de PyPI a NPM

> **Audiencia:** Agente de IA encargado de reimplementar la librería `naturalsql` (actualmente en Python/PyPI) como un paquete TypeScript/JavaScript publicado en NPM.  
> **Versión original de referencia:** `naturalsql 1.2.3` (Python 3.10+)  
> **Objetivo:** Producir `naturalsql` en NPM con paridad funcional completa, misma arquitectura y API pública equivalente.

---

## Índice

1. [Resumen Ejecutivo](#1-resumen-ejecutivo)
2. [Análisis de la Librería Original](#2-análisis-de-la-librería-original)
3. [Mapa de Módulos Python → TypeScript](#3-mapa-de-módulos-python--typescript)
4. [Equivalencias de Dependencias](#4-equivalencias-de-dependencias)
5. [Estructura del Proyecto NPM](#5-estructura-del-proyecto-npm)
6. [Configuración del Proyecto (package.json, tsconfig, build)](#6-configuración-del-proyecto-packagejson-tsconfig-build)
7. [Implementación Módulo por Módulo](#7-implementación-módulo-por-módulo)
   - 7.1 [utils/config.ts](#71-utilsconfigts)
   - 7.2 [utils/constants.ts](#72-utilsconstantsts)
   - 7.3 [utils/prompt.ts](#73-utilspromptts)
   - 7.4 [sql/connection.ts](#74-sqlconnectionts)
   - 7.5 [sql/schemaExtractor.ts](#75-sqlschemaextractorts)
   - 7.6 [sql/queryExecutor.ts](#76-sqlqueryexecutorts)
   - 7.7 [vector/providers/base.ts](#77-vectorprovidersbasetts)
   - 7.8 [vector/providers/local.ts](#78-vectorproviderslocalts)
   - 7.9 [vector/providers/gemini.ts](#79-vectorprovidersgeminits)
   - 7.10 [vector/stores/base.ts](#710-vectorstoresbasetts)
   - 7.11 [vector/stores/chromaStore.ts](#711-vectorstoreschomastorets)
   - 7.12 [vector/stores/sqliteStore.ts](#712-vectorstoressqlitestorets)
   - 7.13 [vector/factory.ts](#713-vectorfactoryts)
   - 7.14 [controller/vectorManager.ts](#714-controllervectormanagerts)
   - 7.15 [api.ts (NaturalSQL class)](#715-apits-naturalsql-class)
   - 7.16 [index.ts (entry point)](#716-indexts-entry-point)
8. [Tests con Vitest](#8-tests-con-vitest)
9. [Publicación en NPM](#9-publicación-en-npm)
10. [CI/CD con GitHub Actions](#10-cicd-con-github-actions)
11. [Diferencias y Decisiones Clave Python → TypeScript](#11-diferencias-y-decisiones-clave-python--typescript)
12. [Checklist Final](#12-checklist-final)

---

## 1. Resumen Ejecutivo

`naturalsql` es una librería ligera que:

1. Se conecta a una base de datos SQL (PostgreSQL, MySQL, SQLite, SQL Server).
2. Extrae el esquema (tablas y columnas) usando drivers nativos.
3. Convierte el esquema en embeddings vectoriales usando modelos locales (`sentence-transformers`) o la API de Google Gemini.
4. Almacena los embeddings en una base vectorial (ChromaDB o SQLite).
5. Permite búsqueda semántica: dada una pregunta en lenguaje natural, devuelve las tablas más relevantes del esquema.
6. Ofrece un helper para construir prompts listos para usar con cualquier LLM.

**Principio de diseño clave:** cero dependencias en el núcleo. Todas las dependencias son opcionales (se cargan de forma dinámica según la combinación configurada).

---

## 2. Análisis de la Librería Original

### Estructura de directorios Python

```
naturalsql/
├── __init__.py                          # Exporta NaturalSQL, AppConfig
├── api.py                               # Clase NaturalSQL (punto de entrada)
├── controller/
│   └── controllervector.py             # Clase VectorManager
├── sql/
│   ├── sqlconecctions.py               # Clase Connection (drivers DB)
│   ├── sqlschema.py                    # Clase SQLSchemaExtractor
│   └── sqlquerys.py                    # Clase QuerysConsult (solo SELECT)
├── utils/
│   ├── config.py                       # Dataclass AppConfig (inmutable)
│   ├── constans.py                     # CONNECTION_QLS, IGNORE_TABLE
│   └── prompt.py                       # Función build_prompt
└── vector/
    ├── factory.py                      # Fábrica de providers y stores
    ├── providers/
    │   ├── base.py                     # ABC EmbeddingProvider
    │   ├── local.py                    # LocalSentenceTransformersProvider
    │   └── gemini.py                   # GeminiEmbeddingProvider
    └── stores/
        ├── base.py                     # ABC VectorStore
        ├── chroma_store.py             # ChromaVectorStore
        └── sqlite_store.py             # SQLiteVectorStore
```

### Flujo de datos

```
[DB URL] → Connection → SQLSchemaExtractor → formated_for_ia()
                                                    ↓
                               EmbeddingProvider.embed_documents()
                                                    ↓
                                        VectorStore.upsert()
                                                    ↓
[Pregunta NL] → EmbeddingProvider.embed_query() → VectorStore.query()
                                                    ↓
                                  [tablas relevantes filtradas]
                                                    ↓
                              build_prompt() → [prompt para LLM]
```

### Combinaciones soportadas

| `vector_backend` | `embedding_provider` | Dependencias npm equivalentes |
|------------------|----------------------|-------------------------------|
| `chroma`         | `local`              | `chromadb`, `@xenova/transformers` |
| `sqlite`         | `local`              | `@xenova/transformers` |
| `sqlite`         | `gemini`             | `@google/generative-ai` |
| `chroma`         | `gemini`             | `chromadb`, `@google/generative-ai` |

---

## 3. Mapa de Módulos Python → TypeScript

| Python | TypeScript |
|--------|-----------|
| `naturalsql/api.py` → `NaturalSQL` | `src/api.ts` → `NaturalSQL` |
| `naturalsql/controller/controllervector.py` → `VectorManager` | `src/controller/vectorManager.ts` → `VectorManager` |
| `naturalsql/sql/sqlconecctions.py` → `Connection` | `src/sql/connection.ts` → `Connection` |
| `naturalsql/sql/sqlschema.py` → `SQLSchemaExtractor` | `src/sql/schemaExtractor.ts` → `SQLSchemaExtractor` |
| `naturalsql/sql/sqlquerys.py` → `QuerysConsult` | `src/sql/queryExecutor.ts` → `QueryExecutor` |
| `naturalsql/utils/config.py` → `AppConfig` | `src/utils/config.ts` → `AppConfig` |
| `naturalsql/utils/constans.py` | `src/utils/constants.ts` |
| `naturalsql/utils/prompt.py` → `build_prompt` | `src/utils/prompt.ts` → `buildPrompt` |
| `naturalsql/vector/factory.py` | `src/vector/factory.ts` |
| `naturalsql/vector/providers/base.py` → `EmbeddingProvider` (ABC) | `src/vector/providers/base.ts` → `EmbeddingProvider` (abstract) |
| `naturalsql/vector/providers/local.py` | `src/vector/providers/local.ts` |
| `naturalsql/vector/providers/gemini.py` | `src/vector/providers/gemini.ts` |
| `naturalsql/vector/stores/base.py` → `VectorStore` (ABC) | `src/vector/stores/base.ts` → `VectorStore` (abstract) |
| `naturalsql/vector/stores/chroma_store.py` | `src/vector/stores/chromaStore.ts` |
| `naturalsql/vector/stores/sqlite_store.py` | `src/vector/stores/sqliteStore.ts` |
| `naturalsql/__init__.py` | `src/index.ts` |

---

## 4. Equivalencias de Dependencias

### Drivers de base de datos

| Python (PyPI) | JavaScript (NPM) | Notas |
|---------------|------------------|-------|
| `psycopg2-binary` | `pg` + `@types/pg` | Driver oficial PostgreSQL para Node.js |
| `pymysql` | `mysql2` | Promesas nativas, compatible con MySQL y MariaDB |
| `sqlite3` (stdlib) | `better-sqlite3` + `@types/better-sqlite3` | API síncrona, sin callbacks |
| `pyodbc` | `mssql` | Driver MSSQL para Node.js; usa `tedious` internamente |

### Embeddings

| Python (PyPI) | JavaScript (NPM) | Notas |
|---------------|------------------|-------|
| `sentence-transformers` (modelo `all-MiniLM-L6-v2`) | `@xenova/transformers` | Transformers.js; corre modelos ONNX en Node.js |
| `google-genai` | `@google/generative-ai` | SDK oficial de Google para Gemini |

### Vector stores

| Python (PyPI) | JavaScript (NPM) | Notas |
|---------------|------------------|-------|
| `chromadb` | `chromadb` | Cliente JS oficial de Chroma (v1.x) |
| `sqlite3` (stdlib) | `better-sqlite3` | Mismo paquete que el driver DB |

### Utilitarios

| Python (stdlib) | JavaScript/Node.js |
|-----------------|-------------------|
| `os.path` | `node:path`, `node:fs` |
| `urllib.parse` | `node:url` (clase `URL`) |
| `dataclasses.dataclass(frozen=True)` | TypeScript `readonly` interface o `Object.freeze()` |
| `typing.ABC` | TypeScript `abstract class` |
| `typing.Literal` | TypeScript union type `"a" \| "b"` |
| `numpy` | Operaciones nativas de arrays JS (no se necesita dependencia) |
| `json` (stdlib) | `JSON.parse` / `JSON.stringify` (nativo) |
| `re` (stdlib) | RegExp nativo de JS |
| `math` (stdlib) | `Math.*` nativo de JS |

### Build y desarrollo

| Python | NPM/Node.js |
|--------|------------|
| `setuptools` + `wheel` | `tsup` (empaquetador CJS+ESM basado en esbuild) |
| `pyproject.toml` | `package.json` |
| `pytest` | `vitest` |
| `requirements-dev.txt` | `devDependencies` en `package.json` |
| `MANIFEST.in` | campo `files` en `package.json` |

---

## 5. Estructura del Proyecto NPM

```
naturalsql/                         ← raíz del proyecto NPM
├── src/
│   ├── index.ts                    ← entry point público
│   ├── api.ts                      ← clase NaturalSQL
│   ├── controller/
│   │   └── vectorManager.ts
│   ├── sql/
│   │   ├── connection.ts
│   │   ├── schemaExtractor.ts
│   │   └── queryExecutor.ts
│   ├── utils/
│   │   ├── config.ts
│   │   ├── constants.ts
│   │   └── prompt.ts
│   └── vector/
│       ├── factory.ts
│       ├── providers/
│       │   ├── base.ts
│       │   ├── local.ts
│       │   └── gemini.ts
│       └── stores/
│           ├── base.ts
│           ├── chromaStore.ts
│           └── sqliteStore.ts
├── tests/
│   ├── api.test.ts
│   ├── schemaExtractor.test.ts
│   ├── sqliteStore.test.ts
│   ├── vectorManager.test.ts
│   └── prompt.test.ts
├── dist/                           ← generado por tsup (NO commitear)
├── package.json
├── tsconfig.json
├── tsup.config.ts
├── vitest.config.ts
├── .npmignore
├── .gitignore
├── LICENSE
└── README.md
```

---

## 6. Configuración del Proyecto (package.json, tsconfig, build)

### 6.1 `package.json`

```json
{
  "name": "naturalsql",
  "version": "1.2.3",
  "description": "Generate SQL from natural language using DB schema + vector retrieval",
  "author": "NaturalSQL Contributors",
  "license": "Apache-2.0",
  "keywords": [
    "sql", "llm", "ai", "vector", "embeddings", "postgresql",
    "mysql", "sqlite", "chromadb", "gemini", "text-to-sql"
  ],
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  },
  "files": [
    "dist",
    "LICENSE",
    "README.md"
  ],
  "scripts": {
    "build": "tsup",
    "dev": "tsup --watch",
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src tests --ext .ts",
    "prepublishOnly": "npm run build && npm run typecheck"
  },
  "peerDependencies": {
    "@google/generative-ai": ">=0.21.0",
    "@xenova/transformers": ">=2.17.0",
    "better-sqlite3": ">=9.0.0",
    "chromadb": ">=1.9.0",
    "mssql": ">=11.0.0",
    "mysql2": ">=3.0.0",
    "pg": ">=8.0.0"
  },
  "peerDependenciesMeta": {
    "@google/generative-ai": { "optional": true },
    "@xenova/transformers": { "optional": true },
    "better-sqlite3": { "optional": true },
    "chromadb": { "optional": true },
    "mssql": { "optional": true },
    "mysql2": { "optional": true },
    "pg": { "optional": true }
  },
  "devDependencies": {
    "@google/generative-ai": "^0.21.0",
    "@types/better-sqlite3": "^7.6.13",
    "@types/node": "^22.0.0",
    "@types/pg": "^8.11.6",
    "@xenova/transformers": "^2.17.2",
    "better-sqlite3": "^9.6.0",
    "chromadb": "^1.9.7",
    "mysql2": "^3.11.0",
    "pg": "^8.13.0",
    "tsup": "^8.3.0",
    "typescript": "^5.7.0",
    "vitest": "^2.1.0"
  },
  "engines": {
    "node": ">=18.0.0"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/JosephAnderson234/NaturalSQL.git"
  }
}
```

> **Nota:** Se usan `peerDependencies` con `optional: true` para replicar el comportamiento de `[project.optional-dependencies]` en Python. El usuario instala sólo lo que necesita según su combinación.

### 6.2 `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### 6.3 `tsup.config.ts`

```typescript
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm", "cjs"],
  dts: true,
  splitting: false,
  sourcemap: true,
  clean: true,
  target: "node18",
  outDir: "dist",
});
```

### 6.4 `vitest.config.ts`

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    globals: true,
    testTimeout: 30_000,
  },
});
```

### 6.5 `.npmignore`

```
src/
tests/
tsconfig.json
tsup.config.ts
vitest.config.ts
.eslintrc*
*.test.ts
.github/
```

---

## 7. Implementación Módulo por Módulo

> Todos los archivos van en `src/`. La implementación es TypeScript estricto con `async/await`.  
> Los imports usan extensión `.js` (requerido por ESM con `moduleResolution: NodeNext`).

---

### 7.1 `utils/config.ts`

**Python original:** `naturalsql/utils/config.py`

```python
@dataclass(frozen=True)
class AppConfig:
    db_url: str | None
    db_type: str
    ...
```

**TypeScript:**

```typescript
// src/utils/config.ts

export type VectorBackend = "chroma" | "sqlite";
export type EmbeddingProviderType = "local" | "gemini";

export interface AppConfig {
  readonly dbUrl: string | null;
  readonly dbType: string;
  readonly normalizeEmbeddings: boolean;
  readonly device: string;
  readonly vectorBackend: VectorBackend;
  readonly embeddingProvider: EmbeddingProviderType;
  readonly geminiApiKey: string | null;
  readonly geminiEmbeddingModel: string;
  readonly vectorDistanceThreshold: number;
}

export function createConfig(params: AppConfig): AppConfig {
  return Object.freeze({ ...params });
}
```

**Notas de migración:**
- `dataclass(frozen=True)` → `Object.freeze()` aplicado en `createConfig`.
- Los nombres van en `camelCase` según convención JS (ej. `db_url` → `dbUrl`).
- `str | None` de Python → `string | null` en TypeScript.
- `Literal["chroma", "sqlite"]` → type alias union `"chroma" | "sqlite"`.

---

### 7.2 `utils/constants.ts`

**Python original:** `naturalsql/utils/constans.py`

```typescript
// src/utils/constants.ts

export const CONNECTION_TEMPLATES: Record<string, string> = {
  postgresql: "postgresql://{user}:{password}@{host}:{port}/{database}",
  mysql: "mysql://{user}:{password}@{host}:{port}/{database}",
  sqlite: "sqlite:///{database}",
  sqlserver:
    "mssql://{user}:{password}@{host}:{port}/{database}?driver=ODBC+Driver+17+for+SQL+Server",
};

export const IGNORE_TABLE: ReadonlySet<string> = new Set([
  // Migrations / versioning
  "alembic_version",
  "django_migrations",
  "schema_migrations",
  "flyway_schema_history",
  "__efmigrationshistory",

  // Task queues
  "celery",
  "celery_taskmeta",
  "celery_tasksetmeta",

  // Vector store metadata
  "langchain_pg_collection",
  "langchain_pg_embedding",
  "vector_store",
  "embeddings",

  // SQLite internals
  "sqlite_sequence",
]);
```

---

### 7.3 `utils/prompt.ts`

**Python original:** `naturalsql/utils/prompt.py`

```typescript
// src/utils/prompt.ts

/**
 * Builds an LLM prompt using relevant tables and the user's question.
 *
 * @param relevantTables - Array of table description strings from the vector search.
 * @param userQuestion   - The natural language question to answer with SQL.
 * @returns A formatted prompt string ready to send to any LLM.
 */
export function buildPrompt(
  relevantTables: string[],
  userQuestion: string
): string {
  const context = relevantTables.join("\n\n");

  return `
You are an expert SQL assistant. Use the following database schema to write a SQL query.

### Schema:
${context}

### Rules:
- Only return the SQL query, no explanations.
- Use the table and column names exactly as defined in the schema.

### Question:
${userQuestion}

### SQL Query:
`.trim();
}
```

---

### 7.4 `sql/connection.ts`

**Python original:** `naturalsql/sql/sqlconecctions.py`

En Python se usa `psycopg2`, `pymysql`, `sqlite3`, `pyodbc` de forma síncrona.  
En Node.js todos los drivers son asíncronos excepto `better-sqlite3` (que es síncrono).  
El patrón `DB-API 2.0` de Python se reemplaza por interfaces ad-hoc para cada driver.

```typescript
// src/sql/connection.ts

import { URL } from "node:url";
import type { AppConfig } from "../utils/config.js";
import { CONNECTION_TEMPLATES } from "../utils/constants.js";

export type DbEngine = "postgresql" | "mysql" | "sqlite" | "sqlserver";

/** Generic raw connection handle returned after connect(). */
// eslint-disable-next-line @typescript-eslint/no-explicit-any
export type RawConnection = any;

export class Connection {
  private connectionString: string;
  public connection: RawConnection = null;

  private constructor(connectionString: string) {
    this.connectionString = connectionString;
  }

  static fromConfig(config: AppConfig): Connection {
    if (!config.dbUrl || !config.dbType) {
      throw new Error("dbUrl and dbType are required in AppConfig.");
    }
    const dbType = config.dbType.toLowerCase() as DbEngine;
    if (!(dbType in CONNECTION_TEMPLATES)) {
      throw new Error(
        `Unsupported dbType: "${config.dbType}". ` +
          `Supported: ${Object.keys(CONNECTION_TEMPLATES).join(", ")}`
      );
    }
    return new Connection(config.dbUrl);
  }

  async connect(): Promise<RawConnection> {
    if (this.connection) return this.connection;
    if (!this.connectionString) throw new Error("No connectionString provided.");

    const parsed = new URL(this.connectionString);
    const scheme = parsed.protocol.replace(":", "").toLowerCase();
    const baseScheme = scheme.split("+")[0];

    const connectors: Record<DbEngine, () => Promise<RawConnection>> = {
      postgresql: () => this._connectPostgreSQL(parsed),
      mysql: () => this._connectMySQL(parsed),
      sqlite: () => this._connectSQLite(parsed),
      sqlserver: () => this._connectSQLServer(parsed),
    };

    const key = (baseScheme === "mssql" ? "sqlserver" : baseScheme) as DbEngine;
    const connector = connectors[key];
    if (!connector) throw new Error(`Unsupported engine in URL: ${scheme}`);

    this.connection = await connector();
    return this.connection;
  }

  private async _connectPostgreSQL(parsed: URL): Promise<RawConnection> {
    const { Client } = await import("pg");
    const client = new Client({
      host: parsed.hostname,
      port: parsed.port ? parseInt(parsed.port) : 5432,
      user: decodeURIComponent(parsed.username),
      password: decodeURIComponent(parsed.password),
      database: parsed.pathname.replace(/^\//, ""),
    });
    await client.connect();
    return client;
  }

  private async _connectMySQL(parsed: URL): Promise<RawConnection> {
    const mysql = await import("mysql2/promise");
    const conn = await mysql.createConnection({
      host: parsed.hostname,
      port: parsed.port ? parseInt(parsed.port) : 3306,
      user: decodeURIComponent(parsed.username),
      password: decodeURIComponent(parsed.password),
      database: parsed.pathname.replace(/^\//, ""),
    });
    return conn;
  }

  private async _connectSQLite(parsed: URL): Promise<RawConnection> {
    // better-sqlite3 is synchronous — wrap in a simple object
    const Database = (await import("better-sqlite3")).default;
    let dbPath = parsed.hostname + parsed.pathname;
    if (!dbPath || dbPath === "/") dbPath = ":memory:";
    const db = new Database(dbPath);
    return db;
  }

  private async _connectSQLServer(parsed: URL): Promise<RawConnection> {
    const mssql = await import("mssql");
    const driverParam =
      new URLSearchParams(parsed.search).get("driver") ??
      "ODBC Driver 17 for SQL Server";
    const pool = await mssql.connect({
      server: parsed.hostname,
      port: parsed.port ? parseInt(parsed.port) : 1433,
      user: decodeURIComponent(parsed.username),
      password: decodeURIComponent(parsed.password),
      database: parsed.pathname.replace(/^\//, ""),
      options: {
        trustServerCertificate: true,
        driver: driverParam,
      },
    });
    return pool;
  }

  async disconnect(): Promise<void> {
    if (!this.connection) return;
    try {
      if (typeof this.connection.end === "function") {
        await this.connection.end(); // pg client
      } else if (typeof this.connection.close === "function") {
        this.connection.close(); // better-sqlite3
      } else if (typeof this.connection.destroy === "function") {
        await this.connection.destroy(); // mysql2 pool
      }
    } finally {
      this.connection = null;
    }
  }
}
```

**Notas de migración:**
- `psycopg2.connect()` → `new pg.Client()` con `await client.connect()`.
- `pymysql.connect()` → `mysql2.createConnection()` (promise API).
- `sqlite3.connect()` → `new Database(path)` de `better-sqlite3` (síncrono).
- `pyodbc.connect()` → `mssql.connect()`.
- Todos son `async/await` excepto `better-sqlite3`.
- Los drivers se importan dinámicamente (`await import(...)`) para replicar los lazy imports de Python.

---

### 7.5 `sql/schemaExtractor.ts`

**Python original:** `naturalsql/sql/sqlschema.py`

```typescript
// src/sql/schemaExtractor.ts

import { IGNORE_TABLE } from "../utils/constants.js";
import type { RawConnection } from "./connection.js";

export type SchemaMap = Record<string, Array<[string, string]>>;

export class SQLSchemaExtractor {
  private connection: RawConnection;
  private dbType: string;

  constructor(connection: RawConnection, dbType: string = "postgresql") {
    this.connection = connection;
    this.dbType = dbType.trim().toLowerCase();
  }

  async extractSchema(): Promise<SchemaMap> {
    const extractors: Record<string, () => Promise<SchemaMap>> = {
      postgresql: () => this._extractPostgreSQL(),
      mysql: () => this._extractMySQL(),
      sqlserver: () => this._extractSQLServer(),
      sqlite: () => this._extractSQLite(),
    };

    const extractor = extractors[this.dbType];
    if (!extractor) {
      throw new Error(
        `Unsupported engine: "${this.dbType}". ` +
          `Use one of: ${Object.keys(extractors).join(", ")}`
      );
    }
    return extractor();
  }

  private async _extractPostgreSQL(): Promise<SchemaMap> {
    const result = await this.connection.query(`
      SELECT table_name, column_name, data_type
      FROM information_schema.columns
      WHERE table_schema = 'public'
      ORDER BY table_name, ordinal_position
    `);
    return this._parseInformationSchema(
      result.rows as [string, string, string][]
    );
  }

  private async _extractMySQL(): Promise<SchemaMap> {
    const [rows] = await this.connection.execute(`
      SELECT table_name, column_name, data_type
      FROM information_schema.columns
      WHERE table_schema = DATABASE()
      ORDER BY table_name, ordinal_position
    `);
    return this._parseInformationSchema(rows as [string, string, string][]);
  }

  private async _extractSQLServer(): Promise<SchemaMap> {
    const result = await this.connection.request().query(`
      SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE
      FROM INFORMATION_SCHEMA.COLUMNS
      WHERE TABLE_SCHEMA = 'dbo'
      ORDER BY TABLE_NAME, ORDINAL_POSITION
    `);
    return this._parseInformationSchema(
      result.recordset.map(
        (r: Record<string, string>) =>
          [r.TABLE_NAME, r.COLUMN_NAME, r.DATA_TYPE] as [string, string, string]
      )
    );
  }

  private _extractSQLite(): Promise<SchemaMap> {
    // better-sqlite3 is synchronous
    const tablesResult = this.connection
      .prepare(`
        SELECT name FROM sqlite_master
        WHERE type = 'table' AND name NOT LIKE 'sqlite_%'
        ORDER BY name
      `)
      .all() as Array<{ name: string }>;

    const schema: SchemaMap = {};
    for (const { name: tableName } of tablesResult) {
      if (IGNORE_TABLE.has(tableName.toLowerCase())) continue;

      const columns = this.connection
        .prepare(`PRAGMA table_info("${tableName}")`)
        .all() as Array<{ name: string; type: string }>;

      schema[tableName] = columns.map((col) => [col.name, col.type || "TEXT"]);
    }
    return Promise.resolve(schema);
  }

  private _parseInformationSchema(
    rows: Array<[string, string, string] | Record<string, string>>
  ): SchemaMap {
    const schema: SchemaMap = {};
    for (const row of rows) {
      const [tableName, columnName, dataType] = Array.isArray(row)
        ? row
        : [
            (row as Record<string, string>).table_name,
            (row as Record<string, string>).column_name,
            (row as Record<string, string>).data_type,
          ];

      if (IGNORE_TABLE.has(tableName.toLowerCase())) continue;
      if (!schema[tableName]) schema[tableName] = [];
      schema[tableName].push([columnName, dataType]);
    }
    return schema;
  }

  formatForAI(schema: SchemaMap): string[] {
    return Object.entries(schema).map(([table, columns]) => {
      const columnDescriptions = columns
        .map(([col, dtype]) => `${col} (${dtype})`)
        .join(", ");
      return `Table name: ${table}. It has the following columns: ${columnDescriptions}`;
    });
  }
}
```

**Notas de migración:**
- Python `cursor.fetchall()` → en `pg`: `result.rows`; en `mysql2`: primer elemento del array destructurado `[rows]`.
- `PRAGMA table_info()` de SQLite se mantiene igual, pero se usa `better-sqlite3`'s `.prepare().all()`.
- `formated_for_ia()` en Python → `formatForAI()` en TypeScript (camelCase).
- SQLite usa API síncrona; se envuelve en `Promise.resolve()` para mantener la interfaz uniforme.

---

### 7.6 `sql/queryExecutor.ts`

**Python original:** `naturalsql/sql/sqlquerys.py`

```typescript
// src/sql/queryExecutor.ts

import type { RawConnection } from "./connection.js";

export type QueryResult = {
  columns: string[];
  rows: unknown[][];
} | null;

export class QueryExecutor {
  private connection: RawConnection;

  constructor(connection: RawConnection) {
    this.connection = connection;
  }

  private cleanSql(rawQuery: string): string {
    return rawQuery.replace(/```sql|```/gi, "").trim();
  }

  async executeQuery(query: string): Promise<QueryResult> {
    const cleaned = this.cleanSql(query);
    const sql = cleaned.trimEnd().replace(/;$/, "");

    if (sql.includes(";")) {
      throw new Error("Multiple SQL statements are not allowed.");
    }
    if (!/^\s*select\b/i.test(sql)) {
      throw new Error("Only read-only SELECT queries are allowed.");
    }

    // pg
    if (typeof this.connection.query === "function" && !this.connection.prepare) {
      const result = await this.connection.query(sql);
      const columns = result.fields?.map((f: { name: string }) => f.name) ?? [];
      return { columns, rows: result.rows };
    }

    // mysql2
    if (typeof this.connection.execute === "function") {
      const [rows, fields] = await this.connection.execute(sql);
      const columns = (fields as Array<{ name: string }>).map((f) => f.name);
      return {
        columns,
        rows: (rows as Record<string, unknown>[]).map((r) => Object.values(r)),
      };
    }

    // better-sqlite3 (sync)
    if (typeof this.connection.prepare === "function") {
      const stmt = this.connection.prepare(sql);
      const rows = stmt.all() as Record<string, unknown>[];
      if (!rows.length) return null;
      const columns = Object.keys(rows[0]);
      return { columns, rows: rows.map((r) => Object.values(r)) };
    }

    throw new Error("Unsupported connection type in QueryExecutor.");
  }
}
```

---

### 7.7 `vector/providers/base.ts`

**Python original:** `naturalsql/vector/providers/base.py`

```typescript
// src/vector/providers/base.ts

export abstract class EmbeddingProvider {
  /** Embed a list of documents (for indexing). */
  abstract embedDocuments(documents: string[]): Promise<number[][]>;

  /** Embed a single query string (for search). */
  abstract embedQuery(query: string): Promise<number[]>;
}
```

**Notas de migración:**
- `ABC` de Python → `abstract class` de TypeScript.
- `@abstractmethod` → keyword `abstract` en TypeScript.
- Todos los métodos son `async` (retornan `Promise`).

---

### 7.8 `vector/providers/local.ts`

**Python original:** `naturalsql/vector/providers/local.py`  
**Modelo:** `all-MiniLM-L6-v2` (384 dimensiones)

```typescript
// src/vector/providers/local.ts

import { EmbeddingProvider } from "./base.js";

export class LocalTransformersProvider extends EmbeddingProvider {
  private pipeline: unknown = null;
  private normalizeEmbeddings: boolean;

  constructor(options: { normalizeEmbeddings?: boolean } = {}) {
    super();
    this.normalizeEmbeddings = options.normalizeEmbeddings ?? true;
  }

  private async getPipeline(): Promise<unknown> {
    if (this.pipeline) return this.pipeline;

    let transformers;
    try {
      transformers = await import("@xenova/transformers");
    } catch {
      throw new Error(
        "Missing dependency: @xenova/transformers. " +
          'Install with: npm install @xenova/transformers'
      );
    }

    this.pipeline = await transformers.pipeline(
      "feature-extraction",
      "Xenova/all-MiniLM-L6-v2"
    );
    return this.pipeline;
  }

  async embedDocuments(documents: string[]): Promise<number[][]> {
    const pipe = await this.getPipeline() as (
      text: string,
      options: Record<string, unknown>
    ) => Promise<{ data: Float32Array }>;

    const results: number[][] = [];
    for (const doc of documents) {
      const output = await pipe(doc, {
        pooling: "mean",
        normalize: this.normalizeEmbeddings,
      });
      results.push(Array.from(output.data));
    }
    return results;
  }

  async embedQuery(query: string): Promise<number[]> {
    const [embedding] = await this.embedDocuments([query]);
    return embedding;
  }
}
```

**Notas de migración:**
- `SentenceTransformer("all-MiniLM-L6-v2")` → `pipeline("feature-extraction", "Xenova/all-MiniLM-L6-v2")` de `@xenova/transformers`.
- `@xenova/transformers` descarga el modelo ONNX automáticamente en el primer uso (~25 MB, se cachea en `~/.cache/huggingface`).
- El parámetro `device` de Python (cpu/cuda) no aplica directamente en Node.js; `@xenova/transformers` usa WASM o WebGPU. Para GPU en Node.js se puede usar `@huggingface/inference` con la API Inference.
- `model.encode(docs, normalize_embeddings=True)` → `pipe(text, { pooling: "mean", normalize: true })`.

---

### 7.9 `vector/providers/gemini.ts`

**Python original:** `naturalsql/vector/providers/gemini.py`

```typescript
// src/vector/providers/gemini.ts

import { EmbeddingProvider } from "./base.js";

export class GeminiEmbeddingProvider extends EmbeddingProvider {
  private genAI: unknown;
  private model: string;

  constructor(options: { apiKey: string; model?: string }) {
    super();
    if (!options.apiKey) {
      throw new Error("Gemini API key is required for GeminiEmbeddingProvider.");
    }
    this.model = options.model ?? "models/text-embedding-004";
    this._initClient(options.apiKey);
  }

  private _initClient(apiKey: string): void {
    // Validation only; lazy import on first use
    if (!apiKey) throw new Error("Gemini API Key is required.");
    // Store apiKey for lazy init
    (this as unknown as Record<string, unknown>)._apiKey = apiKey;
  }

  private async getGenAI(): Promise<unknown> {
    if (this.genAI) return this.genAI;

    let googleGenAI;
    try {
      googleGenAI = await import("@google/generative-ai");
    } catch {
      throw new Error(
        "Missing dependency: @google/generative-ai. " +
          'Install with: npm install @google/generative-ai'
      );
    }

    const apiKey = (this as unknown as Record<string, unknown>)._apiKey as string;
    this.genAI = new googleGenAI.GoogleGenerativeAI(apiKey);
    return this.genAI;
  }

  async embedDocuments(documents: string[]): Promise<number[][]> {
    const genAI = await this.getGenAI() as {
      getGenerativeModel: (opts: Record<string, unknown>) => {
        batchEmbedContents: (req: unknown) => Promise<{ embeddings: Array<{ values: number[] }> }>;
      };
    };

    const model = genAI.getGenerativeModel({ model: this.model });

    try {
      const result = await model.batchEmbedContents({
        requests: documents.map((text) => ({
          content: { parts: [{ text }] },
          taskType: "RETRIEVAL_DOCUMENT",
        })),
      });
      return result.embeddings.map((e) => e.values);
    } catch (err) {
      throw new Error(`Failed to embed documents with Gemini: ${err}`);
    }
  }

  async embedQuery(query: string): Promise<number[]> {
    const genAI = await this.getGenAI() as {
      getGenerativeModel: (opts: Record<string, unknown>) => {
        embedContent: (req: unknown) => Promise<{ embedding: { values: number[] } }>;
      };
    };

    const model = genAI.getGenerativeModel({ model: this.model });

    try {
      const result = await model.embedContent({
        content: { parts: [{ text: query }] },
        taskType: "RETRIEVAL_QUERY",
      });
      return result.embedding.values;
    } catch (err) {
      throw new Error(`Failed to embed query with Gemini: ${err}`);
    }
  }
}
```

**Notas de migración:**
- `google.genai.Client(api_key=...)` → `new GoogleGenerativeAI(apiKey)` de `@google/generative-ai`.
- `client.models.embed_content(model, contents, config=EmbedContentConfig(task_type="RETRIEVAL_DOCUMENT"))` → `model.batchEmbedContents({ requests: [...] })`.
- El SDK JS de Gemini difiere ligeramente de la versión Python; revisar la [documentación oficial](https://ai.google.dev/api/embeddings).

---

### 7.10 `vector/stores/base.ts`

**Python original:** `naturalsql/vector/stores/base.py`

```typescript
// src/vector/stores/base.ts

export abstract class VectorStore {
  /** Add or update documents with their IDs and embeddings. */
  abstract upsert(
    documents: string[],
    ids: string[],
    embeddings: number[][]
  ): Promise<void>;

  /** Query for top-n similar documents. Returns [documents, distances]. */
  abstract query(
    queryEmbedding: number[],
    nResults: number
  ): Promise<[string[], number[]]>;

  /** Return the count of documents stored. */
  abstract count(): Promise<number>;

  /** Reset (delete all documents) the store. */
  abstract reset(): Promise<void>;
}
```

---

### 7.11 `vector/stores/chromaStore.ts`

**Python original:** `naturalsql/vector/stores/chroma_store.py`

```typescript
// src/vector/stores/chromaStore.ts

import { VectorStore } from "./base.js";

export class ChromaVectorStore extends VectorStore {
  private client: unknown = null;
  private collection: unknown = null;
  private storagePath: string;
  private collectionName: string;
  private resetOnStart: boolean;

  constructor(options: {
    storagePath: string;
    collectionName?: string;
    resetOnStart?: boolean;
  }) {
    super();
    this.storagePath = options.storagePath;
    this.collectionName = options.collectionName ?? "db_schema";
    this.resetOnStart = options.resetOnStart ?? false;
  }

  private async getCollection(): Promise<unknown> {
    if (this.collection) return this.collection;

    let chromadb;
    try {
      chromadb = await import("chromadb");
    } catch {
      throw new Error(
        "Missing dependency: chromadb. " +
          'Install with: npm install chromadb'
      );
    }

    this.client = new chromadb.ChromaClient({
      path: this.storagePath,
    });

    if (this.resetOnStart) {
      await this.reset();
    }

    this.collection = await (this.client as {
      getOrCreateCollection: (opts: Record<string, unknown>) => Promise<unknown>;
    }).getOrCreateCollection({
      name: this.collectionName,
      embeddingFunction: null,
    });

    return this.collection;
  }

  async upsert(
    documents: string[],
    ids: string[],
    embeddings: number[][]
  ): Promise<void> {
    const col = await this.getCollection() as {
      upsert: (opts: Record<string, unknown>) => Promise<void>;
    };
    await col.upsert({ documents, ids, embeddings });
  }

  async query(
    queryEmbedding: number[],
    nResults: number
  ): Promise<[string[], number[]]> {
    const col = await this.getCollection() as {
      query: (opts: Record<string, unknown>) => Promise<{
        documents: string[][];
        distances: number[][];
      }>;
    };
    const results = await col.query({
      queryEmbeddings: [queryEmbedding],
      nResults,
    });
    return [results.documents[0], results.distances[0]];
  }

  async count(): Promise<number> {
    const col = await this.getCollection() as { count: () => Promise<number> };
    return col.count();
  }

  async reset(): Promise<void> {
    if (!this.client) return;
    try {
      await (this.client as {
        deleteCollection: (name: string) => Promise<void>;
      }).deleteCollection(this.collectionName);
    } catch {
      // Collection may not exist; ignore error
    }
    this.collection = null;
  }
}
```

**Notas de migración:**
- `chromadb.PersistentClient(path=storage_path)` → `new ChromaClient({ path: storagePath })`.
- La API del cliente JS de ChromaDB es prácticamente idéntica a la Python, con nombres en camelCase.
- `embedding_function=None` → `embeddingFunction: null`.

---

### 7.12 `vector/stores/sqliteStore.ts`

**Python original:** `naturalsql/vector/stores/sqlite_store.py`

```typescript
// src/vector/stores/sqliteStore.ts

import { createRequire } from "node:module";
import { mkdirSync, existsSync } from "node:fs";
import { join } from "node:path";
import { VectorStore } from "./base.js";

type BetterSQLite3Database = {
  prepare: (sql: string) => {
    run: (...args: unknown[]) => void;
    all: () => unknown[];
    get: () => unknown;
  };
  exec: (sql: string) => void;
  close: () => void;
};

export class SQLiteVectorStore extends VectorStore {
  private db: BetterSQLite3Database | null = null;
  private storagePath: string;
  private tableName: string;
  private resetOnStart: boolean;

  constructor(options: {
    storagePath: string;
    tableName?: string;
    resetOnStart?: boolean;
  }) {
    super();
    this.storagePath = options.storagePath;
    this.tableName = options.tableName ?? "vectors";
    this.resetOnStart = options.resetOnStart ?? false;
  }

  private async getDb(): Promise<BetterSQLite3Database> {
    if (this.db) return this.db;

    let Database: new (path: string) => BetterSQLite3Database;
    try {
      // better-sqlite3 uses CommonJS
      const req = createRequire(import.meta.url);
      Database = req("better-sqlite3");
    } catch {
      throw new Error(
        "Missing dependency: better-sqlite3. " +
          'Install with: npm install better-sqlite3'
      );
    }

    if (!existsSync(this.storagePath)) {
      mkdirSync(this.storagePath, { recursive: true });
    }

    const dbPath = join(this.storagePath, "vectors.db");
    this.db = new Database(dbPath);

    if (this.resetOnStart) {
      this.db.exec(`DROP TABLE IF EXISTS ${this.tableName}`);
    }

    this.db.exec(`
      CREATE TABLE IF NOT EXISTS ${this.tableName} (
        id TEXT PRIMARY KEY,
        content TEXT,
        embedding TEXT
      )
    `);

    return this.db;
  }

  async upsert(
    documents: string[],
    ids: string[],
    embeddings: number[][]
  ): Promise<void> {
    const db = await this.getDb();
    const stmt = db.prepare(`
      INSERT OR REPLACE INTO ${this.tableName} (id, content, embedding)
      VALUES (?, ?, ?)
    `);
    for (let i = 0; i < documents.length; i++) {
      stmt.run(ids[i], documents[i], JSON.stringify(embeddings[i]));
    }
  }

  async query(
    queryEmbedding: number[],
    nResults: number
  ): Promise<[string[], number[]]> {
    const db = await this.getDb();
    const rows = db
      .prepare(`SELECT content, embedding FROM ${this.tableName}`)
      .all() as Array<{ content: string; embedding: string }>;

    if (!rows.length) return [[], []];

    const normQuery = this._norm(queryEmbedding);
    const results: Array<[string, number]> = rows.map(({ content, embedding }) => {
      const vec: number[] = JSON.parse(embedding);
      const normVec = this._norm(vec);

      let distance = 1.0;
      if (normQuery !== 0 && normVec !== 0) {
        const similarity = this._dot(queryEmbedding, vec) / (normQuery * normVec);
        distance = 1 - similarity;
      }
      return [content, distance];
    });

    results.sort((a, b) => a[1] - b[1]);
    const topN = results.slice(0, nResults);
    return [topN.map((r) => r[0]), topN.map((r) => r[1])];
  }

  async count(): Promise<number> {
    const db = await this.getDb();
    const row = db
      .prepare(`SELECT COUNT(*) as cnt FROM ${this.tableName}`)
      .get() as { cnt: number };
    return row.cnt;
  }

  async reset(): Promise<void> {
    const db = await this.getDb();
    db.exec(`DELETE FROM ${this.tableName}`);
  }

  private _dot(a: number[], b: number[]): number {
    return a.reduce((sum, ai, i) => sum + ai * (b[i] ?? 0), 0);
  }

  private _norm(v: number[]): number {
    return Math.sqrt(v.reduce((sum, x) => sum + x * x, 0));
  }
}
```

**Notas de migración:**
- `import sqlite3` de Python stdlib → `better-sqlite3` de NPM (API síncrona casi idéntica).
- La similitud coseno se implementa en JS puro, eliminando la dependencia en `numpy`.
- `better-sqlite3` es un módulo CJS; se importa con `createRequire` para compatibilidad con ESM.
- La distancia coseno `1 - cosine_similarity` se mantiene igual.

---

### 7.13 `vector/factory.ts`

**Python original:** `naturalsql/vector/factory.py`

```typescript
// src/vector/factory.ts

import type { AppConfig } from "../utils/config.js";
import type { EmbeddingProvider } from "./providers/base.js";
import type { VectorStore } from "./stores/base.js";

export async function createEmbeddingProvider(
  config: AppConfig
): Promise<EmbeddingProvider> {
  if (config.embeddingProvider === "local") {
    const { LocalTransformersProvider } = await import(
      "./providers/local.js"
    );
    return new LocalTransformersProvider({
      normalizeEmbeddings: config.normalizeEmbeddings,
    });
  }
  if (config.embeddingProvider === "gemini") {
    const { GeminiEmbeddingProvider } = await import(
      "./providers/gemini.js"
    );
    return new GeminiEmbeddingProvider({
      apiKey: config.geminiApiKey!,
      model: config.geminiEmbeddingModel,
    });
  }
  throw new Error(`Unknown embedding provider: ${config.embeddingProvider}`);
}

export async function createVectorStore(
  config: AppConfig,
  storagePath: string,
  reset = false
): Promise<VectorStore> {
  if (config.vectorBackend === "chroma") {
    const { ChromaVectorStore } = await import("./stores/chromaStore.js");
    return new ChromaVectorStore({ storagePath, resetOnStart: reset });
  }
  if (config.vectorBackend === "sqlite") {
    const { SQLiteVectorStore } = await import("./stores/sqliteStore.js");
    return new SQLiteVectorStore({ storagePath, resetOnStart: reset });
  }
  throw new Error(`Unknown vector backend: ${config.vectorBackend}`);
}
```

---

### 7.14 `controller/vectorManager.ts`

**Python original:** `naturalsql/controller/controllervector.py`

```typescript
// src/controller/vectorManager.ts

import { existsSync, statSync } from "node:fs";
import { join } from "node:path";
import type { AppConfig } from "../utils/config.js";
import type { EmbeddingProvider } from "../vector/providers/base.js";
import type { VectorStore } from "../vector/stores/base.js";
import {
  createEmbeddingProvider,
  createVectorStore,
} from "../vector/factory.js";

export class VectorManager {
  private provider: EmbeddingProvider;
  private store: VectorStore;
  private config: AppConfig;

  private constructor(
    config: AppConfig,
    provider: EmbeddingProvider,
    store: VectorStore
  ) {
    this.config = config;
    this.provider = provider;
    this.store = store;
  }

  static async create(options: {
    storagePath: string;
    forceReset: boolean;
    config: AppConfig;
  }): Promise<VectorManager> {
    const { storagePath, forceReset, config } = options;
    const provider = await createEmbeddingProvider(config);
    const store = await createVectorStore(config, storagePath, forceReset);
    return new VectorManager(config, provider, store);
  }

  /**
   * Check whether an indexed collection already exists at storagePath.
   * Returns the count of indexed documents, or 0 if none exist.
   */
  static async collectionExists(storagePath: string): Promise<number> {
    // Check ChromaDB
    const chromaDbPath = join(storagePath, "chroma.sqlite3");
    if (existsSync(chromaDbPath)) {
      try {
        const chromadb = await import("chromadb");
        const client = new chromadb.ChromaClient({ path: storagePath });
        const collection = await client.getCollection({ name: "db_schema" });
        const count = await collection.count();
        if (count > 0) return count;
      } catch {
        // Collection doesn't exist or ChromaDB not installed
      }
    }

    // Check SQLite vector store
    const sqlitePath = join(storagePath, "vectors.db");
    if (existsSync(sqlitePath)) {
      try {
        const { createRequire } = await import("node:module");
        const req = createRequire(import.meta.url);
        const Database = req("better-sqlite3");
        const db = new Database(sqlitePath);
        const row = db
          .prepare("SELECT COUNT(*) as cnt FROM vectors")
          .get() as { cnt: number } | undefined;
        db.close();
        if (row && row.cnt > 0) return row.cnt;
      } catch {
        // Table doesn't exist or better-sqlite3 not installed
      }
    }

    return 0;
  }

  async indexTables(tablesList: string[]): Promise<void> {
    if (!tablesList.length) return;

    const ids = tablesList.map((t) => `table::${t}`);
    const embeddings = await this.provider.embedDocuments(tablesList);
    await this.store.upsert(tablesList, ids, embeddings);
  }

  async searchRelevantTables(request: string, limit = 3): Promise<string[]> {
    const queryEmbedding = await this.provider.embedQuery(request);
    const [documents, distances] = await this.store.query(queryEmbedding, limit);

    const threshold = this.config.vectorDistanceThreshold;
    return documents.filter((_, i) => distances[i] <= threshold);
  }
}
```

**Notas de migración:**
- En Python `VectorManager.__init__` es síncrono porque `create_embedding_provider` y `create_vector_store` no son `async`.
- En TypeScript, los imports dinámicos son `async`, así que el constructor no puede ser `async`. Se usa el patrón **Static factory method** (`VectorManager.create()`).
- `collection_exists()` es un `@staticmethod` en Python → método `static async` en TypeScript.

---

### 7.15 `api.ts` (NaturalSQL class)

**Python original:** `naturalsql/api.py`

```typescript
// src/api.ts

import type { VectorBackend, EmbeddingProviderType, AppConfig } from "./utils/config.js";
import { createConfig } from "./utils/config.js";
import { Connection } from "./sql/connection.js";
import { SQLSchemaExtractor } from "./sql/schemaExtractor.js";
import { VectorManager } from "./controller/vectorManager.js";

export interface NaturalSQLOptions {
  dbUrl?: string | null;
  dbType?: string;
  normalizeEmbeddings?: boolean;
  device?: string;
  vectorBackend?: VectorBackend;
  embeddingProvider?: EmbeddingProviderType;
  geminiApiKey?: string | null;
  geminiEmbeddingModel?: string;
  vectorDistanceThreshold?: number;
}

export interface BuildVectorDbResult {
  storagePath: string;
  indexedDocuments: number;
  fromCache: boolean;
}

type BackendProviderCombo = `${VectorBackend}:${EmbeddingProviderType}`;

const COMBINATION_REQUIREMENTS: Record<
  BackendProviderCombo,
  { extra: string; packages: string[] }
> = {
  "chroma:local": {
    extra: "chromadb + @xenova/transformers",
    packages: ["chromadb", "@xenova/transformers"],
  },
  "sqlite:local": {
    extra: "@xenova/transformers",
    packages: ["@xenova/transformers"],
  },
  "sqlite:gemini": {
    extra: "@google/generative-ai",
    packages: ["@google/generative-ai"],
  },
  "chroma:gemini": {
    extra: "chromadb + @google/generative-ai",
    packages: ["chromadb", "@google/generative-ai"],
  },
};

export class NaturalSQL {
  private config: AppConfig;
  private _vectorManager: VectorManager | null = null;
  private _vmStoragePath: string | null = null;

  constructor(options: NaturalSQLOptions = {}) {
    const vectorBackend = (
      (options.vectorBackend ?? "chroma").trim().toLowerCase()
    ) as VectorBackend;
    const embeddingProvider = (
      (options.embeddingProvider ?? "local").trim().toLowerCase()
    ) as EmbeddingProviderType;

    // Validate supported combinations
    const comboKey: BackendProviderCombo = `${vectorBackend}:${embeddingProvider}`;
    if (!(comboKey in COMBINATION_REQUIREMENTS)) {
      throw new Error(
        `Unsupported combination: (${vectorBackend}, ${embeddingProvider}). ` +
          `Supported: ${Object.keys(COMBINATION_REQUIREMENTS).join(", ")}`
      );
    }

    // Validate Gemini API key
    if (embeddingProvider === "gemini" && !options.geminiApiKey) {
      throw new Error(
        "geminiApiKey is required when embeddingProvider is 'gemini'."
      );
    }
    if (embeddingProvider !== "gemini" && options.geminiApiKey) {
      throw new Error(
        "geminiApiKey should only be provided when embeddingProvider is 'gemini'."
      );
    }

    // Validate distance threshold
    const threshold = options.vectorDistanceThreshold ?? 1.0;
    if (typeof threshold !== "number" || isNaN(threshold)) {
      throw new Error("vectorDistanceThreshold must be a number.");
    }
    if (threshold <= 0) {
      throw new Error("vectorDistanceThreshold must be greater than 0.");
    }

    this.config = createConfig({
      dbUrl: options.dbUrl ?? null,
      dbType: (options.dbType ?? "").trim().toLowerCase(),
      normalizeEmbeddings: options.normalizeEmbeddings ?? true,
      device: (options.device ?? "cpu").trim().toLowerCase(),
      vectorBackend,
      embeddingProvider,
      geminiApiKey: options.geminiApiKey ?? null,
      geminiEmbeddingModel:
        options.geminiEmbeddingModel ?? "models/text-embedding-004",
      vectorDistanceThreshold: threshold,
    });
  }

  private _invalidateCache(): void {
    this._vectorManager = null;
    this._vmStoragePath = null;
  }

  private async _getVectorManager(storagePath: string): Promise<VectorManager> {
    if (this._vectorManager && this._vmStoragePath === storagePath) {
      return this._vectorManager;
    }
    this._vectorManager = await VectorManager.create({
      storagePath,
      forceReset: false,
      config: this.config,
    });
    this._vmStoragePath = storagePath;
    return this._vectorManager;
  }

  /**
   * Build (or reuse) the vector database from the connected SQL schema.
   *
   * @param storagePath  - Directory where the vector store is persisted.
   * @param forcedReset  - If true, drops existing data and re-indexes.
   * @returns            - Object with storagePath, indexedDocuments, fromCache.
   */
  async buildVectorDb(
    storagePath = "./metadata_vdb",
    forcedReset = false
  ): Promise<BuildVectorDbResult> {
    if (!this.config.dbUrl) {
      throw new Error(
        "buildVectorDb requires dbUrl. Provide it in the NaturalSQL constructor."
      );
    }

    if (forcedReset) {
      this._invalidateCache();
    }

    if (!forcedReset) {
      const existingCount = await VectorManager.collectionExists(storagePath);
      if (existingCount > 0) {
        return { storagePath, indexedDocuments: existingCount, fromCache: true };
      }
    }

    const conn = Connection.fromConfig(this.config);
    await conn.connect();

    try {
      const extractor = new SQLSchemaExtractor(
        conn.connection,
        this.config.dbType
      );
      const schema = await extractor.extractSchema();
      const formatted = extractor.formatForAI(schema);

      const vm = await VectorManager.create({
        storagePath,
        forceReset: forcedReset,
        config: this.config,
      });
      await vm.indexTables(formatted);

      this._vectorManager = vm;
      this._vmStoragePath = storagePath;

      return {
        storagePath,
        indexedDocuments: formatted.length,
        fromCache: false,
      };
    } finally {
      await conn.disconnect();
    }
  }

  /**
   * Search for tables relevant to a natural language request.
   *
   * @param request      - Natural language query.
   * @param storagePath  - Where the vector store is located.
   * @param limit        - Max number of tables to return.
   * @returns            - Array of table description strings.
   */
  async search(
    request: string,
    storagePath = "./metadata_vdb",
    limit = 3
  ): Promise<string[]> {
    const vm = await this._getVectorManager(storagePath);
    return vm.searchRelevantTables(request, limit);
  }
}
```

---

### 7.16 `index.ts` (entry point)

**Python original:** `naturalsql/__init__.py`

```typescript
// src/index.ts

export { NaturalSQL } from "./api.js";
export type { NaturalSQLOptions, BuildVectorDbResult } from "./api.js";
export type { AppConfig, VectorBackend, EmbeddingProviderType } from "./utils/config.js";
export { buildPrompt } from "./utils/prompt.js";

// Advanced exports (optional, for users who need lower-level access)
export { EmbeddingProvider } from "./vector/providers/base.js";
export { VectorStore } from "./vector/stores/base.js";
export { SQLSchemaExtractor } from "./sql/schemaExtractor.js";
export { QueryExecutor } from "./sql/queryExecutor.js";
export { Connection } from "./sql/connection.js";
export { VectorManager } from "./controller/vectorManager.js";
```

---

## 8. Tests con Vitest

### 8.1 `tests/prompt.test.ts` (sin dependencias externas)

```typescript
import { describe, it, expect } from "vitest";
import { buildPrompt } from "../src/utils/prompt.js";

describe("buildPrompt", () => {
  it("includes the user question", () => {
    const result = buildPrompt(["Table name: users."], "List all users");
    expect(result).toContain("List all users");
  });

  it("includes the schema context", () => {
    const result = buildPrompt(
      ["Table name: orders. It has the following columns: id (integer)"],
      "Show orders"
    );
    expect(result).toContain("Table name: orders");
  });

  it("joins multiple tables with blank lines", () => {
    const result = buildPrompt(["Table A", "Table B"], "query");
    expect(result).toContain("Table A\n\nTable B");
  });
});
```

### 8.2 `tests/schemaExtractor.test.ts` (SQLite, con `better-sqlite3`)

```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import Database from "better-sqlite3";
import { SQLSchemaExtractor } from "../src/sql/schemaExtractor.js";

describe("SQLSchemaExtractor (SQLite)", () => {
  let db: ReturnType<typeof Database>;

  beforeAll(() => {
    db = new Database(":memory:");
    db.exec(`
      CREATE TABLE users (id INTEGER, name TEXT, email TEXT);
      CREATE TABLE orders (id INTEGER, user_id INTEGER, amount REAL);
    `);
  });

  afterAll(() => db.close());

  it("extracts schema for all tables", async () => {
    const extractor = new SQLSchemaExtractor(db, "sqlite");
    const schema = await extractor.extractSchema();
    expect(Object.keys(schema)).toContain("users");
    expect(Object.keys(schema)).toContain("orders");
  });

  it("formats schema for AI correctly", async () => {
    const extractor = new SQLSchemaExtractor(db, "sqlite");
    const schema = await extractor.extractSchema();
    const formatted = extractor.formatForAI(schema);
    expect(formatted[0]).toMatch(/^Table name:/);
    expect(formatted[0]).toContain("columns:");
  });
});
```

### 8.3 `tests/sqliteStore.test.ts`

```typescript
import { describe, it, expect, beforeAll } from "vitest";
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { SQLiteVectorStore } from "../src/vector/stores/sqliteStore.js";

describe("SQLiteVectorStore", () => {
  let store: SQLiteVectorStore;
  let tmpPath: string;

  beforeAll(async () => {
    tmpPath = mkdtempSync(`${tmpdir()}/naturalsql-test-`);
    store = new SQLiteVectorStore({ storagePath: tmpPath, resetOnStart: true });
  });

  it("upserts and queries documents", async () => {
    const docs = ["Table name: users. columns: id, name"];
    const ids = ["table::users"];
    const embeddings = [[0.1, 0.2, 0.3]];

    await store.upsert(docs, ids, embeddings);

    const count = await store.count();
    expect(count).toBe(1);

    const [resultDocs, distances] = await store.query([0.1, 0.2, 0.3], 1);
    expect(resultDocs).toHaveLength(1);
    expect(distances[0]).toBeCloseTo(0, 5); // Same vector = distance ≈ 0
  });

  it("resets the store", async () => {
    await store.reset();
    expect(await store.count()).toBe(0);
  });
});
```

### 8.4 `tests/api.test.ts` (con mocks)

```typescript
import { describe, it, expect, vi } from "vitest";
import { NaturalSQL } from "../src/api.js";

describe("NaturalSQL constructor", () => {
  it("throws for unsupported combination", () => {
    expect(
      () =>
        new NaturalSQL({
          vectorBackend: "chroma",
          embeddingProvider: "unknown" as never,
        })
    ).toThrow(/unsupported combination/i);
  });

  it("throws when geminiApiKey missing for gemini provider", () => {
    expect(
      () =>
        new NaturalSQL({
          embeddingProvider: "gemini",
          vectorBackend: "sqlite",
        })
    ).toThrow(/geminiApiKey is required/i);
  });

  it("throws for invalid distanceThreshold", () => {
    expect(
      () => new NaturalSQL({ vectorDistanceThreshold: -1 })
    ).toThrow(/greater than 0/i);
  });

  it("throws for buildVectorDb when dbUrl is not provided", async () => {
    const nsql = new NaturalSQL({ vectorBackend: "sqlite", embeddingProvider: "local" });
    await expect(nsql.buildVectorDb()).rejects.toThrow(/requires dbUrl/i);
  });
});
```

---

## 9. Publicación en NPM

### 9.1 Pre-requisitos

```bash
# 1. Crear cuenta en npmjs.com si no tienes una
# 2. Iniciar sesión
npm login

# 3. Verificar identidad
npm whoami
```

### 9.2 Preparar el paquete

```bash
# Instalar dependencias de desarrollo
npm install

# Compilar TypeScript → dist/
npm run build

# Verificar tipos
npm run typecheck

# Ejecutar tests
npm test

# Verificar qué archivos se incluirán en el paquete
npm pack --dry-run
```

### 9.3 Versionado semántico

```bash
# Patch (bug fix): 1.2.3 → 1.2.4
npm version patch

# Minor (nueva funcionalidad compatible): 1.2.3 → 1.3.0
npm version minor

# Major (breaking change): 1.2.3 → 2.0.0
npm version major
```

### 9.4 Publicar

```bash
# Publicar en npmjs.com (paquete público)
npm publish --access public

# Para una versión beta/pre-release
npm publish --tag beta
```

### 9.5 Verificar publicación

```bash
# Ver información del paquete publicado
npm info naturalsql

# Instalar en otro proyecto para verificar
npm install naturalsql
```

### 9.6 Instalación por combinación (equivalente a extras de Python)

Los usuarios instalan sólo lo que necesitan:

```bash
# ChromaDB + embeddings locales
npm install naturalsql chromadb @xenova/transformers

# SQLite + embeddings locales (más ligero)
npm install naturalsql @xenova/transformers better-sqlite3

# SQLite + Gemini API
npm install naturalsql @google/generative-ai better-sqlite3

# ChromaDB + Gemini API
npm install naturalsql chromadb @google/generative-ai

# Con soporte PostgreSQL
npm install naturalsql pg @xenova/transformers

# Con soporte MySQL
npm install naturalsql mysql2 @xenova/transformers
```

---

## 10. CI/CD con GitHub Actions

Crear `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run typecheck

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

Crear `.github/workflows/publish.yml`:

```yaml
name: Publish to NPM

on:
  release:
    types: [created]

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: "https://registry.npmjs.org"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run tests
        run: npm test

      - name: Publish
        run: npm publish --access public --provenance
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

> **Nota:** Configurar el secreto `NPM_TOKEN` en GitHub → Settings → Secrets → Actions.  
> El flag `--provenance` genera provenance attestation (recomendado para seguridad en NPM ≥ 2024).

---

## 11. Diferencias y Decisiones Clave Python → TypeScript

### 11.1 Asincronismo

| Python | TypeScript/Node.js |
|--------|--------------------|
| API síncrona (DB-API 2.0) | Todo es `async/await` |
| `sqlite3.connect()` síncrono | `better-sqlite3` es síncrono; se puede envolver en `Promise.resolve()` |
| `dataclass` inmutable | `Object.freeze()` + interfaz `readonly` |

### 11.2 Constructores asíncronos

En Python, `__init__` puede ejecutar lógica de configuración sin async.  
En TypeScript, los constructores **no pueden ser async**. Para módulos que requieren imports dinámicos (lazy loading), usar el patrón **Static factory method**:

```typescript
// ❌ No funciona en TS
class Foo {
  constructor() {
    await import("something"); // Error: cannot await in constructor
  }
}

// ✅ Static factory
class Foo {
  static async create(): Promise<Foo> {
    const mod = await import("something");
    return new Foo(mod);
  }
}
```

### 11.3 Módulos opcionales

| Python | TypeScript |
|--------|-----------|
| `[project.optional-dependencies]` en `pyproject.toml` | `peerDependencies` con `optional: true` en `package.json` |
| `try: __import__("mod")` con mensaje de error útil | `try { await import("mod") } catch { throw new Error("Install...") }` |

### 11.4 Tipos

| Python | TypeScript |
|--------|-----------|
| `str \| None` | `string \| null` |
| `list[str]` | `string[]` |
| `dict[str, list[tuple[str, str]]]` | `Record<string, Array<[string, string]>>` |
| `tuple[list[str], list[float]]` | `[string[], number[]]` |
| `Literal["a", "b"]` | `"a" \| "b"` |
| `ABC` + `@abstractmethod` | `abstract class` + método `abstract` |
| `@dataclass(frozen=True)` | `readonly interface` + `Object.freeze()` |
| `frozenset` | `ReadonlySet` |

### 11.5 Paths y sistema de archivos

| Python | TypeScript/Node.js |
|--------|--------------------|
| `os.path.join(a, b)` | `path.join(a, b)` de `node:path` |
| `os.path.exists(p)` | `existsSync(p)` de `node:fs` |
| `os.makedirs(p, exist_ok=True)` | `mkdirSync(p, { recursive: true })` de `node:fs` |

### 11.6 Modelo de embeddings local

| Python | TypeScript |
|--------|-----------|
| `SentenceTransformer("all-MiniLM-L6-v2")` de `sentence-transformers` | `pipeline("feature-extraction", "Xenova/all-MiniLM-L6-v2")` de `@xenova/transformers` |
| Soporte GPU via `device="cuda"` | Soporte GPU via WebGPU (experimental); CPU por defecto |
| Modelo en cache `~/.cache/torch/sentence_transformers/` | Modelo en cache `~/.cache/huggingface/hub/` |

### 11.7 ChromaDB

El cliente JS de ChromaDB (`chromadb`) tiene una API casi idéntica a la Python, pero:
- El cliente JS por defecto espera un servidor Chroma corriendo en `http://localhost:8000`.
- Para almacenamiento persistente local sin servidor, usar `ChromaClient` con `path` en versiones que lo soporten, o migrar a `EphemeralClient`/`PersistentClient` según la versión.
- Revisar la [documentación del cliente JS de ChromaDB](https://docs.trychroma.com/reference/js-client) al momento de implementar.

### 11.8 ESM vs CJS

El proyecto usa `"type": "module"` en `package.json`, lo que significa que todos los archivos `.js` se tratan como ESM. Sin embargo, algunos paquetes (como `better-sqlite3`) son CJS. Para importarlos en ESM:

```typescript
// Para módulos CJS en contexto ESM
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
const Database = require("better-sqlite3");
```

---

## 12. Checklist Final

Usa esta lista para verificar que la migración está completa:

### Estructura

- [ ] Proyecto inicializado con `npm init` o manualmente con `package.json`
- [ ] `tsconfig.json` configurado (target ES2022, NodeNext moduleResolution)
- [ ] `tsup.config.ts` configurado (dual CJS/ESM output)
- [ ] `vitest.config.ts` configurado
- [ ] `.npmignore` creado (excluye `src/`, `tests/`, configs de dev)
- [ ] `.gitignore` actualizado (excluye `dist/`, `node_modules/`)

### Módulos implementados

- [ ] `src/utils/config.ts` — `AppConfig` interface + `createConfig`
- [ ] `src/utils/constants.ts` — `IGNORE_TABLE`, `CONNECTION_TEMPLATES`
- [ ] `src/utils/prompt.ts` — `buildPrompt`
- [ ] `src/sql/connection.ts` — `Connection` con 4 drivers
- [ ] `src/sql/schemaExtractor.ts` — `SQLSchemaExtractor` con 4 engines
- [ ] `src/sql/queryExecutor.ts` — `QueryExecutor` (solo SELECT)
- [ ] `src/vector/providers/base.ts` — `EmbeddingProvider` abstract
- [ ] `src/vector/providers/local.ts` — `LocalTransformersProvider`
- [ ] `src/vector/providers/gemini.ts` — `GeminiEmbeddingProvider`
- [ ] `src/vector/stores/base.ts` — `VectorStore` abstract
- [ ] `src/vector/stores/chromaStore.ts` — `ChromaVectorStore`
- [ ] `src/vector/stores/sqliteStore.ts` — `SQLiteVectorStore` (con cosine similarity nativa)
- [ ] `src/vector/factory.ts` — `createEmbeddingProvider`, `createVectorStore`
- [ ] `src/controller/vectorManager.ts` — `VectorManager` (con static factory)
- [ ] `src/api.ts` — `NaturalSQL` (clase principal)
- [ ] `src/index.ts` — Exportaciones públicas

### Tests

- [ ] `tests/prompt.test.ts` ✅
- [ ] `tests/schemaExtractor.test.ts` ✅
- [ ] `tests/sqliteStore.test.ts` ✅
- [ ] `tests/api.test.ts` (constructor validations) ✅
- [ ] `tests/vectorManager.test.ts` (con mocks de provider/store)

### Validación funcional

- [ ] `npm run build` — Sin errores
- [ ] `npm run typecheck` — Sin errores de tipos
- [ ] `npm test` — Todos los tests pasan
- [ ] Prueba manual con SQLite + local embeddings (`better-sqlite3` + `@xenova/transformers`)
- [ ] Prueba manual con SQLite + Gemini (requiere `GEMINI_API_KEY`)
- [ ] Prueba manual con ChromaDB + local (requiere servidor Chroma)

### NPM

- [ ] `package.json` con todos los campos requeridos (`name`, `version`, `description`, `license`, `exports`, `peerDependencies`)
- [ ] `npm pack --dry-run` — Lista de archivos correcta (solo `dist/`, `README.md`, `LICENSE`)
- [ ] `npm login` completado
- [ ] `npm publish --access public` ejecutado
- [ ] `npm info naturalsql` confirma publicación

### CI/CD

- [ ] `.github/workflows/ci.yml` — Tests en Node 18, 20, 22
- [ ] `.github/workflows/publish.yml` — Auto-publish en GitHub Release
- [ ] Secreto `NPM_TOKEN` configurado en GitHub

---

## Referencias

- [Repositorio Python original](https://github.com/JosephAnderson234/NaturalSQL)
- [Transformers.js (@xenova/transformers)](https://github.com/xenova/transformers.js)
- [node-postgres (pg)](https://node-postgres.com/)
- [mysql2](https://sidorares.github.io/node-mysql2/docs)
- [better-sqlite3](https://github.com/WiseLibs/better-sqlite3)
- [mssql](https://github.com/tediousjs/node-mssql)
- [ChromaDB JS Client](https://docs.trychroma.com/reference/js-client)
- [Google Generative AI JS SDK](https://github.com/google-gemini/generative-ai-js)
- [tsup](https://tsup.egoist.dev/)
- [vitest](https://vitest.dev/)
- [NPM publish docs](https://docs.npmjs.com/cli/v10/commands/npm-publish)
