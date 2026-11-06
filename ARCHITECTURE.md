# Headless Document Engine: architecture

Holds only what Hleb has approved. If something isn't here, it isn't decided: ask Hleb, don't assume. Nothing is built yet.

## What it is

A headless engine, installed as a tool and imported into other projects, that gives document generation one standard. It has no screens of its own: apps, buttons and automations are built on top of it.

- **Goal:** lightweight, agentic, easy to use, compliant, standardised.
- **Scope:** contracts, invoices, reports, account data, accounting books, analytics, výkazy, official forms and filing packages.
- **New document type:** a few new files, never a new script.
- **Purpose:** conflicts and double definitions are removed, not managed.
- **Size:** one maintainer and about 50 users. No complications, an easy and optimised flow.

## The whole picture

```text
ENGINE (one package, command: document-engine)
├── Core            checks data, builds the document, runs connectors
├── Ways in         code, CLI, HTTP: all call the same core functions
├── Data table      every $TAG, one table across all documents
└── Basic library   basic documents and common connectors

LIBRARY (own repo, outside the engine)
├── Data requests   tags it wants added to the data table
├── Doctypes        one per document, each with its own content
├── Sets            several documents built together
└── Connectors      only those it adds
```

## Parts

### Data

One table belongs to the tool, not to libraries. `$CLIENT` is one unique tag, reusable across all documents.

Each tag has:

- **Rules:** data type, allowed lists, AND/OR rules. `$DOBA_TYPE` can only be `urcita` or `neurcita`, so `mixed` is refused.
- **Keywords:** at least 3 unique.
- **Description:** one short sentence.

Keywords and descriptions are written in English or proper names. That is a filling rule and note; the engine doesn't check it.

The values themselves are input that arrives through an endpoint, a form, an app or CLI input. They never live in a library. The engine checks values against the rules, and never whether they are true: whether a client exists isn't its job.

### Content

What the document says. Each item has its own ID and can hold text, images or other special items, with formatting and style rules such as size, bold and paragraph.

Content is linked to its doctype and isn't shared between doctypes. A repeated value such as a company name comes from data: `$COMPANY_NAME`.

Data and content are related by rules. If `$DOBA_TYPE` is `urcita`, the sentence with `$DATA_KONEC_URCITY_DOBY` appears; otherwise it doesn't.

### Doctype

The format of one document, with no separate template layer. It defines:

- where each content item goes, by its ID
- grid and structure
- how the document displays in different formats and when printed
- how endpoints connect to it
- fillable fields and signature places

One doctype is one document combination. If content, data rules or conditions can't express a variation, copy it into a new doctype. Example: an invoice doctype can show different header text for services and for material, with the same data in the same places, and add the required wording for reverse charge.

### Connector

Data in and out, replaceable without touching the core. Examples: PDF and XML output, an ERP or CLI input.

- **Form:** one JavaScript module in a library, loaded into the engine. Library connectors are committed as one bundled file.
- **Heavy tools:** Typst and similar run as child processes started by the connector.
- **Declaration:** `{ "id": "pdf", "api": 1, "kind": "out", "entry": "./index.js" }`
- **Contract:** `run(ctx)`. An output connector returns `{ files }`, an input connector returns `{ data }`.
- **`ctx`:** the standard document, the doctype's format, the output folder, options and a cancel signal.
- **Shipping:** common connectors ship with the engine. A connector is part of a library and never a repo of its own. A library can hold only a connector and installs the same way.

## How a document is generated

1. A call arrives from code, the CLI or an endpoint.
2. An input connector supplies the data, unless the call already carries it.
3. The core checks the data against the tag rules and the doctype. On a problem it stops with a clear error.
4. It chooses the content that applies, using the rules.
5. It puts the data into the content.
6. It creates one standard document file, with content but no look.
7. Each output connector writes its files.
8. It records which versions produced each file.

## Ways in

Code, CLI and HTTP are thin layers over the same core functions, so they give the same result. The core knows nothing about HTTP or terminals.

| Function     | What it does                    |
| ------------ | ------------------------------- |
| `list`       | Lists document types            |
| `describe`   | Shows required data and outputs |
| `check`      | Checks data and writes no file  |
| `build`      | Generates the files             |
| `connectors` | Lists connectors                |

The CLI is `document-engine`, for example `document-engine build hpp --data jan.json --to pdf`. The HTTP server starts with `document-engine serve`. The calling app stays responsible for its screens and users.

## Libraries

"Library" is the only word for this, with no marketplace or catalog concept. Public and private libraries work the same way; the only difference is who can read the repo.

- **Engine and library:** users install the engine, and their library lives outside it in its own repo.
- **Basic library:** built into the engine, so an installed engine can already make basic documents. People pass their data to them, for example step by step in the CLI or through their own form. These are real documents, not demos, and there's no preview build. Every new library points to it by default.
- **Template:** `afframe/document-library` is a public repo, licensed MIT-0, with one example per type. `document-engine library new` downloads it.
- **No data:** a library ships tags such as `$NAME` or `$SUM`, never values.
- **Pinning:** a library is pinned to an exact commit, and the engine never updates it by itself. `library update` runs only on request and checks the library with no data.

### Installing a library

Doctypes, content and connectors go into the register. Each data field is a request to the data table, and is checked against it:

| Field         | What can happen                                            |
| ------------- | ---------------------------------------------------------- |
| Doesn't exist | Add it, or merge it into an existing tag                   |
| Exists        | Merge it into the existing tag, or add it under a new name |

- **Finding matches:** the validator compares names, keywords and descriptions, and flags similarities and references. The check is deliberately loose: missing diacritics and similar spelling details must not defeat it.
- **Applying the answer:** a script applies every merge and rename to the installed library automatically.
- **Record of changes:** the script writes a special file listing what was merged or renamed and why, so a reinstall, a move or a share shows what was changed automatically.
- **Renaming later:** renaming a tag is as easy as creating one. The old name is replaced with the new.

## Server and connector safety

- The server loads only the libraries the project declares. A request can't name a library, path or command.
- `library add` prints the connectors and the pinned commit.
- The engine checks output names: plain name, regular file, inside the output folder, no symlinks.
- Each build gets a fresh private folder, removed afterwards.
- Builds have a timeout through the cancel signal and a concurrency limit, default 1, with a queue.
- Programs the engine starts get a minimal environment.
- The server restarts automatically, because a connector bug can stop it.

A connector is code from its author and runs with the engine's full rights. Installing a library that holds connectors means installing that software.

## Project rules

- **Repo:** `afframe/document-engine`, public. The remote is the source of truth. Commits are prepared on a branch and merged to `main` when Hleb approves. PRs are only for Dependabot, external contributions and releases.
- **License:** MPL-2.0 with a simple CLA: a checkbox in the PR, no bot. Copyright holder: Hleb Tkachenko.
- **Other libraries:** each sets its own license.
- **Public repo content:** never client names or real data. Examples use invented data.
