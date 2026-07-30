# project-secrets-create

Create secret files in `.ci-secrets` (or a custom directory) from secret entries, matching the behavior of a typical workflow step that materializes secrets onto disk for build tools (e.g., Docker build secrets, npm auth files, Maven settings, etc.).

This action:
- requires that the secrets directory is explicitly listed in `.dockerignore`
- creates the directory with restrictive permissions (umask `077`)
- writes each secret entry as a file named `key` containing `value`
- optionally fails if any secret value is empty
- applies ownership (`chown`) to the directory and created files

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `pairs` | yes | — | Multiline list of secret entries. Supports `key=value` (single-line) and `key<<DELIM ... DELIM` (multiline). `key` becomes the filename under the secrets directory. Blank lines and lines beginning with `#` are ignored outside multiline blocks. |
| `dir` | no | `.ci-secrets` | Directory to write secret files into. Must be a relative path and must not contain `..`. |
| `owner` | no | `1000:1000` | Ownership to apply via `chown owner:group` to the directory and files. |
| `fail_on_empty` | no | `true` | If `true`, the action fails when any `value` is empty. Case-insensitive. |

### `pairs` format

- Single-line entry: `filename=value`
- Multiline entry:
  `filename<<DELIM`
  `...value lines...`
  `DELIM`
- `filename` must match: `^[A-Za-z0-9._-]+$`
- Whitespace around the `=` is allowed.
- Everything after the first `=` is treated as the value (including additional `=` characters).
- Leading/trailing whitespace for control lines is trimmed.
- Lines starting with `#` are ignored outside multiline blocks.

Example:

```yaml
pairs: |
  nexus_user=${{ secrets.NEXUS_USER }}
  nexus_password=${{ secrets.NEXUS_PASSWORD }}
  # comment lines are ignored
  npm_token=${{ secrets.NPM_TOKEN }}
```

Multiline value example:

```yaml
pairs: |
  maven_settings_xml<<EOF
  ${{ secrets.MAVEN_SETTINGS_XML }}
  EOF
  npm_token=${{ secrets.NPM_TOKEN }}
```

## Usage

### Basic example

```yaml
- uses: frozen-tapestry/project-secrets-create@v1
  with:
    pairs: |
      nexus_user=${{ secrets.NEXUS_USER }}
      nexus_password=${{ secrets.NEXUS_PASSWORD }}
```

### Custom directory and ownership

```yaml
- uses: frozen-tapestry/project-secrets-create@v1
  with:
    dir: .ci-secrets
    owner: 1001:1001
    pairs: |
      npm_token=${{ secrets.NPM_TOKEN }}
      maven_settings_xml<<EOF
      ${{ secrets.MAVEN_SETTINGS_XML }}
      EOF
```

### Allow empty values (not recommended)

```yaml
- uses: frozen-tapestry/project-secrets-create@v1
  with:
    fail_on_empty: "false"
    pairs: |
      optional_value=${{ secrets.OPTIONAL_VALUE }}
```

## Required `.dockerignore` entry

This action **requires**:

1. a `.dockerignore` file exists at repository root, and
2. the secrets directory is present as an **exact line** in `.dockerignore`.

For the default directory:

```gitignore
.ci-secrets
```

This is a guardrail to reduce accidental inclusion of secret files in Docker build contexts.

## Security notes

* The secrets directory must be a safe relative path. Absolute paths and any path containing `..` are rejected.
* Filenames are validated and restricted to `A-Za-z0-9._-` to reduce the risk of path traversal or unexpected filesystem behavior.
* Files are written with `printf '%s'` (no trailing newline added).
* Directory and files are created under `umask 077` (owner-only by default).
* `chown` runs on the directory and any files that were created.

## License

See [LICENSE](LICENSE).
