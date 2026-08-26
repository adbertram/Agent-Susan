# PDF verification tooling

## Root cause

The `pdfinfo` and `pdftotext` commands are not built into macOS. They are provided by the Poppler package. On this machine, Homebrew was installed at `/opt/homebrew/bin/brew`, but Poppler was not installed, so PDF verification failed with:

```text
/bin/bash: line 2: pdfinfo: command not found
```

## Durable fix applied on 2026-07-09

Installed Poppler with Homebrew:

```bash
brew install poppler
```

Verified installed tools:

```text
/opt/homebrew/bin/pdfinfo
/opt/homebrew/bin/pdftotext
pdfinfo version 26.07.0
pdftotext version 26.07.0
```

The normal Hermes PATH includes `/opt/homebrew/bin`, so no wrapper or PATH change is required for this workspace.

## Standard verification command

Use this form to verify a downloaded PDF without creating a text sidecar file:

```bash
PDF='/absolute/path/to/file.pdf'
pdfinfo "$PDF"
pdftotext "$PDF" - | python3 -c 'import sys; text=sys.stdin.read(); print("pdftotext_chars={}".format(len(text))); print("pdftotext_nonempty={}".format(bool(text.strip())))'
```

For private documents, avoid saving extracted text unless the task requires it. Pipe `pdftotext` to stdout or a parser and print only the needed assertions/counts.

## If the commands are missing in a future session

Run:

```bash
command -v pdfinfo || brew install poppler
command -v pdftotext
```

If Homebrew is unavailable, report the blocker and use a task-local Python extractor only after installing an explicit dependency such as `pypdf` or `pdfminer.six` in an approved project environment.
