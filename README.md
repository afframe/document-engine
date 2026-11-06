# Headless Document Engine

Generate any document from templates and data, through code, a CLI or HTTP endpoints.

You give the engine a document type and data, and it writes the files: contracts, reports, accounting books, financial statements, official forms and filing packages. A new document type needs a few new files, not a new script. The engine has no screens of its own, so it can run on its own or as the backend of any app.

## Status

Design stage. There is no code yet, and everything below describes the planned design, which can still change.

## Planned design

| Layer          | What it is                                                                                    |
| -------------- | --------------------------------------------------------------------------------------------- |
| **Core**       | Checks data, builds the standard document format, loads libraries and connectors              |
| **Ways in**    | Code library, CLI and HTTP server, all calling the same core functions                        |
| **Connectors** | Add-ons that bring data in or write files out, such as PDF, Word, Excel, fillable PDF and XML |
| **Libraries**  | Document types, wording and templates, kept public, private or local                          |

Each document type is made of:

| Part              | What it holds                                                                            |
| ----------------- | ---------------------------------------------------------------------------------------- |
| **Template**      | The look or file layout for one output format                                            |
| **Document type** | Required data, which blocks appear, their order and conditions, and which outputs it has |
| **Text**          | The wording, including grammar such as case, gender and plurals                          |
| **Data**          | The values for one run, passed in at generation                                          |

## License

Copyright 2026 Hleb Tkachenko. Licensed under the [Mozilla Public License 2.0](LICENSE). You can use, change and share the engine, and combine it with your own code under your own terms. If you distribute a changed engine file, you must share that file's source code under the same license.
