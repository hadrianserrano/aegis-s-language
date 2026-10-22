# Repository Guidance

## Scope

This repository currently contains Markdown specifications and guides for the Aegis-S language and toolchain. It does not contain compiler source code, a package manifest, or an automated test suite.

## Working in this repository

- Keep changes focused on the document or feature the user asked about; preserve the existing Markdown structure and terminology where practical.
- Keep filenames aligned with the modular file list in `README.md`; use that list as the repository index.
- Treat descriptions of compiler, runtime, cryptography, formal verification, and security behavior as design proposals unless supported by concrete implementation and validation in the repository. Do not imply that a security property is implemented or proven based only on specification text.
- When editing a specification, keep syntax, terminology, and examples consistent across related documents when the requested scope calls for it. Avoid broad synchronization edits otherwise.
- For documentation-only changes, review the edited sections and links for consistency. There are currently no project build or test commands to run.