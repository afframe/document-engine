# Headless Document Engine: architecture

Holds only what Hleb has approved. If something isn't here, it isn't decided: ask Hleb, don't assume. Nothing is built yet.

## What it is

A headless engine, installed as a tool and imported into other projects, that gives document generation one standard. It has no screens of its own: apps, buttons and automations are built on top of it.

- **Goal:** lightweight, agentic, easy to use, compliant, standardised, deterministic.
- **Scope:** contracts, invoices, reports, account data, accounting books, analytics, výkazy, official forms and filing packages.
- **New document type:** a few new files, never a new script.
- **Purpose:** conflicts and double definitions are removed, not managed.
- **Size:** one maintainer and about 50 users. No complications, an easy and optimised flow.

## The whole picture

```text
form · database · ERP · app · agent · person
        │  { doctype, data, to }
        ▼
ENDPOINT    list · describe · check · build
        ▼
CORE        check → choose content → fill → standard document
        ▼
CONNECTOR   write(ctx), in its own worker
        ▼
PDF · DOCX · XML · send to ERP
```

Three things exist on disk:

```text
TOOL (one package, command: hde)
├── Core and Endpoint
└── Basic library     basic tags, documents and common connectors

LIBRARY (own repo, created or imported)
├── Data              tags it proposes to the data table
├── Doctype           one per document, each with its own content
├── Set               several documents built together
└── Connector         only those it adds

PROJECT (one set per project, shares nothing with other projects)
├── hde.json          libraries it imports
├── hde.lock          exact commits, hashes, merge and rename answers
├── hde.data.json     its additions to the data table
└── .hde/             installed libraries, regenerated, not committed
```

## Parts

### Data

The tool defines the data table and its rules. Each project has exactly one table: the basic library's tags plus the project's additions. `$CLIENT` is one unique tag across all documents of that project. A tag is only standard across projects if the tool ships it.

Each tag has:

- **Rules:** data type, allowed lists, AND/OR rules. `$DOBA_TYPE` can only be `urcita` or `neurcita`, so `mixed` is refused. A rule across two tags applies only in a doctype that uses both.
- **Keywords:** at least 3, distinct inside the tag.
- **Description:** one short sentence.
- **Shape:** a tag can be `group` (named fields, written `$GROUP.field`) or `rows` (a repeating group). It is still one table row, with one set of keywords and one description. An inner field has no keywords: it holds a rule or points at another tag (`"ico": "$ICO"`). Inner field names are compared at install.

Keywords and descriptions are written in English or proper names. That is a filling rule and note; the engine doesn't check it.

The values themselves arrive through Endpoint. They never live in a library. The engine checks values against the rules, and never whether they are true: whether a client exists isn't its job.

### Content

What the document says. Each item has its own ID and can hold text, images or other special items, with formatting and style rules such as size, bold and paragraph.

Content is linked to its doctype and isn't shared between doctypes. A repeated value such as a company name comes from data: `$COMPANY_NAME`.

Data and content are related by rules. If `$DOBA_TYPE` is `urcita`, the sentence with `$DATA_KONEC_URCITY_DOBY` appears; otherwise it doesn't.

Numbers, dates and similar values display through named formats in content, such as `$TOTAL|money` and `$DIC|digits`. One tag holds one value, in one form, and each place formats it as it needs. `$$` prints a literal `$`.

### Doctype

The format of one document, with no separate template layer. It defines:

- which tags it requires and which are optional, by path; the data table has no required flag
- where each content item goes, by its ID, and the rules for showing it
- grid and structure
- how the document displays in different formats and when printed
- fillable fields and signature places
- its default outputs

One doctype is one document combination. If content, data rules or conditions can't express a variation, copy it into a new doctype. Example: an invoice doctype can show different header text for services and for material, with the same data in the same places, and add the required wording for reverse charge.

### Endpoint

The one defined way into the engine, owned by core and the same in every use. A library can't add or change it. Forms, databases and ERPs call it; the engine doesn't connect to them. Pulling data from an ERP is the caller's script:

```bash
erp-pull 123 | hde build invoice --data - --to pdf,erp
```

| Function   | What it does                                                            |
| ---------- | ----------------------------------------------------------------------- |
| `list`     | Lists doctypes and connectors                                           |
| `describe` | Returns a doctype's input as JSON Schema, its outputs and tool versions |
| `check`    | Lists every error at once, with tag path and fix; writes nothing        |
| `build`    | Generates the files and returns them with their record                  |

The same four functions, format and errors are reached by:

| Reached by | How                                                                                                                               |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------- |
| SDK        | `createEngine({ project }).build(doctype, data, { to })`. One instance serves one project, so a SaaS can run one set per customer |
| CLI        | `hde build hpp --data jan.json --to pdf`, also used in scripts. Step-by-step input is built from `describe`                       |
| HTTP       | `hde serve`. Serves only the project it started with                                                                              |
| MCP        | `hde mcp`, with four fixed tools. Its dependency loads only for this command                                                      |

- **Input:** `{ doctype, data, to? }`, in JSON. `build` depends only on this and the project.
- **Result:** `build` returns `{ files: [{ name, bytes }], record }`. The CLI writes to `--out`. HTTP returns JSON with base64 contents, for one file or several.
- **For agents:** every function takes `--json`. `--no-input` never prompts, and nothing prompts without a terminal.
- **Not in Endpoint:** installing and managing libraries are CLI and SDK only, so a web request can't bring in code.

The calling app stays responsible for its screens and users.

### Connector

Output only. It turns the standard document into files or sends it to a system such as an ERP. Core owns the contract; the tool or a library ships it. A connector accepts the standard document, so it works on any doctype; `--to xml` fails clearly if the connector needs something the doctype doesn't have.

- **Form:** one bundled `.mjs` module, `connector/<id>/index.mjs`, exporting `write(ctx)`, with an optional `version()` for `describe` and the record.
- **Declaration:** in `library.json`: `{ "connector": { "erp": { "entry": "connector/erp/index.mjs", "env": ["ERP_ACCESS"] } } }`
- **Returns:** `{ files, receipt? }`. A receipt goes into the record.
- **`ctx`:** the standard document, the doctype's format, `data` (checked values under the connector's own library's tag names), the output folder, `options` from `hde.json`, `env` holding only the declared variables, and a cancel signal, which it must pass to every program it starts.
- **Running:** a fresh worker per call, with a timeout and a memory limit. A crash becomes a build error and doesn't stop the host. The environment is `PATH` plus the declared variables.
- **Heavy tools:** Typst and similar run as child processes started by the connector. Typst is downloaded pinned on first use and verified; a missing tool is a clear error.
- **Shipping:** common connectors ship with the tool. A connector is part of a library and never a repo of its own. A library can hold only a connector and installs the same way.

Example, a complete ERP connector:

```js
export async function write(ctx) {
  const res = await fetch(ctx.options.url, {
    method: 'POST',
    headers: {
      'x-access': ctx.env.ERP_ACCESS,
      'content-type': 'application/json',
    },
    body: JSON.stringify(ctx.document),
  });
  if (!res.ok) throw new Error(`ERP refused: ${res.status}`);
  return { files: [], receipt: await res.text() };
}
```

## How a document is generated

1. A call arrives through Endpoint, carrying the doctype and the data.
2. The core checks the data against the tag rules and the doctype. On a problem it stops and lists every error.
3. It chooses the content that applies, using the rules.
4. It puts the data into the content.
5. It creates one standard document, in JSON, with content but no look.
6. The connectors named by the call, or else the doctype's defaults, run. `check` never runs a connector.
7. It returns the files with a record: engine version, each library's pin and file hash, table hash, lock hash, tool versions and receipts.

These are fixed core steps, not something users define.

**Deterministic:** the same project, input and tool version give the same bytes. The core uses no clock and no network, tools and fonts are pinned, and dates, including the PDF date, come from the call. A connector that sends to a system can repeat what it sends, but a retry sends it again.

## Library

A library works like an app store: people create what they need or import what others made. One tool can hold someone's own invoice library with its own ERP connector, a public Word library, a public PDF analytics library and a connector-only XML library. Public and private libraries work the same way; the only difference is who can read the repo.

```text
library.json
data.json
doctype/<id>/     the doctype and its content
set/<id>.json     a named list of doctypes, built from one set of values
connector/<id>/
```

- **May ship:** doctype, content, proposed data, set, connector.
- **May not ship:** values, secrets, Endpoint code, install scripts.
- **Find:** public libraries tag their GitHub repo `hde-library`, and `hde library search` looks them up, with no server.
- **Create:** `hde library new` downloads `afframe/document-library`, a public repo, licensed MIT-0, with one example per type.
- **Basic library:** built into the tool, so an installed tool can already make basic documents. People pass their data to them, for example step by step in the CLI or through their own form. These are real documents, not demos, and there's no preview build. Every project installs it first.
- **Requests:** a library requests every tag it uses, except shipped ones.
- **Sets:** all members are checked before any file is written. Files are named `<doctype>-<name>`.

### Installing a library

`hde library add` copies doctypes, content and connectors into the project unchanged. Each tag in the library's data is a request to the data table:

| Field                  | What can happen                                                                                                         |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Doesn't exist          | Add it, or merge it into an existing tag                                                                                |
| Exists                 | Merge it into the existing tag, or add it under a new name                                                              |
| Changed by its library | Take the new rule (refused if another library uses the tag and the rule differs for it), merge, or add under a new name |

- **Finding matches:** the validator compares names, keywords and descriptions after lowercasing and removing diacritics, and flags similar ones at medium strictness. A miss is possible, so the person or agent has the final say.
- **No flag:** a field with no similarity flag is added automatically and logged.
- **Same name and rule:** merged automatically and logged.
- **Merge rule:** a merge needs the same type, shape, allowed list and pattern. A group may gain fields but not change one. Two tags of one library can't merge into one. A failed merge is refused with the difference.
- **Other names:** a doctype, set or connector ID already in use is a question whose only answer is rename. A second repo with the same library ID is refused. A shipped connector can't be replaced.
- **Applying the answer:** files stay exactly as pinned. The engine maps library tag names to table names when it loads them, and `hde.lock` records every merge, rename and answer with its reason, so a reinstall, a move or a share shows what changed and why.
- **Without a person:** `--no-input` fails and lists the open questions as data. `--answers file` takes the same format as `hde.lock`. Each answer holds a hash of the rule it answered, and a changed rule is asked again, never added automatically. A keyword or description edit isn't a question.
- **Renaming later:** `hde tag rename` covers tags and inner fields, and is as easy as creating one. It is refused if the new name exists. The old name stays reserved, so a new tag can take it only through a question. A call still sending the old name is refused, and the error names the new tag.
- **Removing:** `hde library remove` drops rows no other library uses.
- **Connector code:** `library add` prints each connector's `env` list and option names with the pin, because installing a connector means installing its author's software.

## Tool and project

They work as Node and npm do. The tool is installed once per Mac, or per repo if a project needs its own version. Libraries belong to a project.

- **Finding the project:** the engine walks up folders to the nearest `hde.json`. With none, it uses the home set in `~/.hde`, which is never merged with a project.
- **Separate projects:** a tool in Afframe, on a desktop and in a local project are three sets that share no data or libraries. This stops accidental sharing; it isn't protection against a malicious connector.
- **What to commit:** `hde.json`, `hde.lock` and `hde.data.json`. `.hde/` is regenerated, and `~/.hde/cache` holds downloads.
- **Pinning:** a git library is pinned to an exact commit, and a folder library to a content hash. The engine never updates one by itself. `hde library update` runs only on request and checks the library with no data.
- **Working on a library:** import it by path. It is re-imported automatically when it changes, with no commit.
- **Deploy:** `hde install --frozen` installs exactly what `hde.lock` says, asks nothing, and fails on a mismatch. `hde serve` and `createEngine` verify the same at start.
- **Tool updates:** a new version can ship a tag a project already defined. `hde install` runs the normal check on it, and `--frozen` fails.

## Server and connector safety

- The server loads only the libraries the project declares. A request can't name a library, path or command.
- The engine checks output names: plain name, regular file, inside the output folder, no symlinks.
- Each build gets a fresh private folder, removed after the result is returned.
- Each engine process has a concurrency limit, default 1, with a queue.
- Connector options holding secrets are refused in `hde.json`. Secrets come only through declared `env`.
- In a SaaS, only the host's own team adds connectors, because a worker doesn't stop a connector reading another customer's files.
- The loader refuses duplicate keys and tags defined twice.

A worker contains crashes, not bad intent: a malicious connector can still read project files and use the network. Pinning and showing it at install are the protection.

## Project rules

- **Name:** the product is Headless Document Engine, never just HDE. `hde` is only the short command, and the repo stays `afframe/document-engine`.
- **Part names:** singular, no trailing "s": Data, Content, Doctype, Endpoint, Connector, Library, Set.
- **Repo:** public. The remote is the source of truth. Commits are prepared on a branch and merged to `main` when Hleb approves. PRs are only for Dependabot, external contributions and releases.
- **License:** MPL-2.0 with a simple CLA: a checkbox in the PR, no bot. Copyright holder: Hleb Tkachenko.
- **Other libraries:** each sets its own license.
- **Public repo content:** never client names or real data. Examples use invented data.
