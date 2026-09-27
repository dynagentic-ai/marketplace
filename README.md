# Dynagentic Claude Code plugin marketplace

Register with:

```
claude plugin marketplace add dynagentic-ai/marketplace
```

Then install:

```
claude plugin install dynagentic-student@dynagentic-connector
claude plugin install dynagentic-staff@dynagentic-connector
```

Each plugin's `source` is an `archive` pointing at its CloudFront-hosted `.plugin` zip, pinned by
`sha256`. When a plugin is republished with new content, this file's `sha256` must be updated to
match, or installs will fail the integrity check.
