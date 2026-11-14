# Pre-commit Hooks Guide


Recommended `.pre-commit-config.yaml`:
```yml
repos:
  # General file hygiene
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: check-merge-conflict
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: check-added-large-files
        args: ["--maxkb=1000"]
      - id: detect-private-key
      - id: debug-statements
      - id: check-case-conflict

  # Python linting and formatting
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.3
    hooks:
      - id: ruff-check
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format

  # Modernize Python syntax
  - repo: https://github.com/asottile/pyupgrade
    rev: v3.20.0
    hooks:
      - id: pyupgrade
        args: [--py311-plus]

  # Security scanning
  - repo: https://github.com/pycqa/bandit
    rev: 1.9.4
    hooks:
      - id: bandit
        args: ["-c", "pyproject.toml"]
        additional_dependencies: ["bandit[toml]"]

  # Spell checking
  - repo: https://github.com/codespell-project/codespell
    rev: v2.4.1
    hooks:
      - id: codespell

  # Prevent committing directly to main
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: no-commit-to-branch
        args: [--branch, main, --branch, master]

  # Validate hook configuration itself
  - repo: meta
    hooks:
      - id: check-hooks-apply
      - id: check-useless-excludes
```


## Libraries & their configs

### Ruff 
Usage: formatting, import sorting, unused imports and variables, undefined names, common Python bugs and bad practices
Example config:
```toml
[tool.ruff]
# Exclude a variety of commonly ignored directories.
exclude = [
    ".bzr",
    ".direnv",
    ".eggs",
    ".git",
    ".git-rewrite",
    ".hg",
    ".ipynb_checkpoints",
    ".mypy_cache",
    ".nox",
    ".pants.d",
    ".pyenv",
    ".pytest_cache",
    ".pytype",
    ".ruff_cache",
    ".svn",
    ".tox",
    ".venv",
    ".vscode",
    "__pypackages__",
    "_build",
    "buck-out",
    "build",
    "dist",
    "node_modules",
    "site-packages",
    "venv",
]

# Same as Black.
line-length = 88
indent-width = 4

# Assume Python 3.11
target-version = "py311"

[tool.ruff.lint]
# select = [...]  # See the Default Rules page for the full listing.
ignore = []

# Allow fix for all enabled rules (when `--fix`) is provided.
fixable = ["ALL"]
unfixable = []

# Allow unused variables when underscore-prefixed.
dummy-variable-rgx = "^(_+|(_+[\\p{XID_Continue}]*[\\p{XID_Continue}--_]+?))$"

[tool.ruff.format]
# Like Black, use double quotes for strings.
quote-style = "double"

# Like Black, indent with spaces, rather than tabs.
indent-style = "space"

# Like Black, respect magic trailing commas.
skip-magic-trailing-comma = false

# Like Black, automatically detect the appropriate line ending.
line-ending = "auto"

# Enable auto-formatting of code examples in docstrings. Markdown,
# reStructuredText code/literal blocks and doctests are all supported.
#
# This is currently disabled by default, but it is planned for this
# to be opt-out in the future.
docstring-code-format = false

# Set the line length limit used when formatting code snippets in
# docstrings.
#
# This only has an effect when the `docstring-code-format` setting is
# enabled.
docstring-code-line-length = "dynamic"

# Exclude files
exclude = ["*.pyi"]
```

### Semgrep
Usage: static application security tester that can flag patterns, find bugs, and enforce code standards.

Basic config `.pre-commit-config.yaml`:
```yml
  - repo: local
    hooks:
      - id: semgrep
        name: Semgrep security and architecture scan
        entry: semgrep --config p/python --config p/owasp-top-ten --config .semgrep
        language: system
        types: [python]
        pass_filenames: false
```

Custom Rule Examples:

**Prevent unsafe HTTP requests**
```yml
rules:
  - id: python.requests-disabled-tls-verification
    languages: [python]
    severity: ERROR
    message: >
      TLS certificate verification is disabled.
      Do not use verify=False in production requests.
    patterns:
      - pattern: requests.$METHOD(..., verify=False, ...)
```
Catches patterns that disable TLS cert verification like:
```python
requests.get(url, verify=False)
requests.post(url, json=data, verify=False)
```

**Prevent unsafe subprocess execution**
```yml
rules:
  - id: python.subprocess-shell-true
    languages: [python]
    severity: ERROR
    message: >
      Avoid shell=True with subprocess.
      Use argument lists instead of shell command strings.
    patterns:
      - pattern: subprocess.$FUNC(..., shell=True, ...)
```

**Prevent unsafe deserialization**
```yml
rules:
  - id: python.unsafe-pickle
    languages: [python]
    severity: ERROR
    message: >
      Avoid deserializing untrusted data with pickle.
    pattern-either:
      - pattern: pickle.loads(...)
      - pattern: pickle.load(...)
      - pattern: dill.loads(...)
      - pattern: dill.load(...)
```

**Prevent insecure Flask template rendering**
```yml
rules:
  - id: flask.unsafe-template-rendering
    languages: [python]
    severity: WARNING
    message: >
      Review render_template_string usage.
      User-controlled template content can lead to SSTI.
    pattern: flask.render_template_string(...)
```

**Prevent sensitive information from being logged**
```yml
rules:
  - id: system.sensitive-data-logging
    languages: [python]
    severity: WARNING
    message: >
      Potentially sensitive client information is being logged.
      Review whether this information should be exposed in logs.
    patterns:
      - pattern-either:
          - pattern: logging.$METHOD(..., ssn, ...)
          - pattern: logging.$METHOD(..., $OBJ.ssn, ...)
          - pattern: logging.$METHOD(..., $OBJ.social_security_number, ...)
          - pattern: logger.$METHOD(..., $OBJ.ssn, ...)
          - pattern: logger.$METHOD(..., $OBJ.dob, ...)
          - pattern: logger.$METHOD(..., $OBJ.address, ...)
          - pattern: logger.$METHOD(..., $OBJ.phone_number, ...)
          - pattern: logger.$METHOD(..., $OBJ.phone, ...)
```

**Prevent hardcoded development credentials**
```yml
rules:
  - id: salix.hardcoded-development-secret
    languages: [python]
    severity: ERROR
    message: >
      Potential hardcoded credential.
      Retrieve secrets from Secret Manager or environment configuration.
    patterns:
      - pattern-either:
          - pattern: $VAR = "..."
          - pattern: $VAR = '...'
      - metavariable-regex:
          metavariable: $VAR
          regex: (?i).*(password|secret|api_key|private_key|access_token).*
```

### Architectural Enforcement Examples
Architecture asummptions
| Layer | Allowed to import | Should not import |
|---|---|---|
| Controller | Processor | DB |
| Processor | Service | Controller, DB |
| Service | DB | Controller, Processor |
| DB | - | Controller, Processor, Service |

**Rule 1: Controllers cannot import services or data stores**
```yml
rules:
  - id: your-system.controller-layer-violation
    languages: [python]
    severity: ERROR
    message: >
      Controllers must communicate through processors.
      Direct service and data-store access is prohibited.
    patterns:
      - pattern-inside: |
          ...
      - pattern-either:
          - pattern: from app.services.$MODULE import ...
          - pattern: import app.services.$MODULE
          - pattern: from app.db.$MODULE import ...
          - pattern: import app.db.$MODULE
    paths:
      include:
        - "**/controllers/**/*.py"
```

**Rule 2: Processors cannot import data stores**
```yml
rules:
  - id: your-system.processor-layer-violation
    languages: [python]
    severity: ERROR
    message: >
      Processors must access persistence through services.
      Do not import data stores directly.
    pattern-either:
      - pattern: from app.db.$MODULE import ...
      - pattern: import app.db.$MODULE
    paths:
      include:
        - "**/processors/**/*.py"
```

**Rule 3: Services cannot import controllers or processors**
```yml
rules:
  - id: salix.service-layer-violation
    languages: [python]
    severity: ERROR
    message: >
      Services must not depend on controllers or processors.
    pattern-either:
      - pattern: from app.controllers.$MODULE import ...
      - pattern: import app.controllers.$MODULE
      - pattern: from app.processors.$MODULE import ...
      - pattern: import app.processors.$MODULE
    paths:
      include:
        - "**/services/**/*.py"
```

**Prevent direct database access outside of the datastore layer**
```yml
rules:
  - id: your-system.database-access-outside-store
    languages: [python]
    severity: ERROR
    message: >
      Database clients should only be initialized or accessed
      through the data-store layer.
    pattern-either:
      - pattern: firestore.Client(...)
      - pattern: google.cloud.firestore.Client(...)
      - pattern: firebase_admin.firestore.client(...)
      - pattern: firebase_admin.storage.bucket(...)
    paths:
      exclude:
        - "**/data_stores/**/*.py"
        - "**/tests/**/*.py"
```

Other rules to consider:
| Rule | What it prevents |
|---|---|
| No Flask imports in processors/services	| HTTP framework leaking into business logic |
| No Firestore queries in services	| Persistence logic bypassing data stores |
| No business logic in controllers	| Controllers becoming bloated |
| No circular layer dependencies	| Architectural coupling |
| No direct external API calls in controllers	| External integrations bypassing services |
| No direct OpenFGA calls outside authorization module	| Inconsistent authorization handling |

