# op-exec

Execute commands with 1Password secrets as environment variables. Similar to `doppler run`, but uses a 1Password item as the config source.

## Usage

```bash
op-exec [options] op://vault/item -- command [args...]
op-exec [options] op://vault/item  # print exports (for debugging)
```

You always pass a **whole item** reference (`op://vault/item`), not a single
field. op-exec reads every field of the item and turns each one into an
environment variable.

## Examples

```bash
op-exec op://Development/my-app-config -- npm start
op-exec op://Work/aws-creds -- aws s3 ls
```

With no command, op-exec prints shell-quoted `export NAME=value` lines (quoted
with `printf %q`) so you can source them into the current shell:

```bash
eval "$(op-exec op://vault/item)"
```

## Installation

```bash
brew install nsheaps/devsetup/op-exec
```

## Requirements

- [1Password CLI](https://developer.1password.com/docs/cli) (`op`)
- `jq`

You must be authenticated to 1Password. op-exec checks this with `op whoami`,
which works both with an interactive session (`op signin`) and with a service
account token (`OP_SERVICE_ACCOUNT_TOKEN`). If neither is available it exits
with an error.

## How it works

### Whole-item → env var mapping

Given `op://vault/item`, op-exec fetches the item and iterates its fields. Every
field whose type is `STRING` or `CONCEALED` becomes an environment variable. All
other field types are ignored.

### Field-label → env-var-name conversion

Field labels are converted into valid environment variable names:

- uppercased,
- spaces and hyphens become underscores,
- any remaining characters outside `[A-Z0-9_]` are stripped.

```text
"API Key"      -> API_KEY
"database-url"  -> DATABASE_URL
```

### Recursive resolution (headline feature)

A field's value can be a literal value **or itself an `op://` reference**. When
the value is a reference, op-exec resolves it with `op read` and repeats: if the
result is *still* an `op://` reference, it follows that one too, and so on. This
lets you build chains where one item points at a field in another item, which in
turn points at yet another.

```text
op://Development/my-app-config  field "DATABASE_URL"
  value: op://Shared/postgres/connection-string   # a reference
    value: op://Vault/rds-prod/url                 # another reference
      value: postgres://user:pass@host:5432/db     # finally a literal
```

`DATABASE_URL` ends up set to the literal at the end of the chain.

Resolution is bounded by a maximum depth of **5** hops. If a value is still an
`op://` reference once that depth is reached, op-exec stops and leaves the raw
reference string as the value (it is not resolved further). A reference that
fails to resolve (empty `op read` result) is likewise left as its raw string.

## Tracking concealed (secret) values

op-exec can record which variables hold secrets so a downstream process can
redact them from logs.

A variable is flagged **concealed** if either:

- its top-level field type is `CONCEALED`, or
- **any hop** in its `op://` resolution chain targets a `CONCEALED` field.

Nuance: a 2-segment cross reference (`op://vault/item`, with no field segment)
points at an entire item, so a single field type cannot be determined for it.
Such a hop is treated conservatively as **non-concealed** — only the top-level
field type or a 3-segment `op://vault/item/field` hop can mark a variable
concealed.

Two flags expose this information:

- `--concealed-file FILE` — **appends** the *names* of concealed variables, one
  per line, to `FILE`. (Names only, no values.)
- `--concealed-kv-file FILE` — writes `NAME=value` pairs for concealed
  variables to `FILE`. The file is **truncated** first and `chmod`ed to `0600`
  since it holds secret values. Multi-line values (e.g. PEM keys) are
  **skipped**, because the file format is one entry per line.

Both files are written before the command is exec'd, so they are populated even
if the child process exits non-zero.

```bash
# Names only — for redaction without exposing values in the file
op-exec --concealed-file /tmp/.env.secrets op://vault/item -- npm start

# NAME=value pairs — for redaction without a separate env lookup
op-exec --concealed-kv-file /tmp/.env.secrets.values op://vault/item -- npm start
```

## Options

| Option | Description |
| --- | --- |
| `-h`, `--help` | Show help and exit. |
| `-d`, `--debug` | Enable debug output on stderr (or set `DEBUG=1`). |
| `--concealed-file FILE` | Append concealed variable **names** (one per line) to `FILE`. |
| `--concealed-kv-file FILE` | Write concealed `NAME=value` pairs to `FILE` (truncated, `chmod 0600`, multi-line values skipped). |

The reference must match `op://vault/item`. Anything after `--` is the command
to exec with the resolved environment.

## Limitations

This is a proof of concept. Known limitations (from the script header):

- **No section handling** — only flat items are supported; fields inside
  sections are not addressed specially.
- **Max recursion depth of 5** — deeper `op://` chains are left unresolved.
- **Only `STRING` and `CONCEALED` field types are exported** — every other field
  type is ignored.

## License

MIT
