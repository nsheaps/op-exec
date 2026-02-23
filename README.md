# op-exec

Execute commands with 1Password secrets as environment variables. Similar to `doppler run`, but uses a 1Password item as the config source.

## Usage

```bash
op-exec op://vault/item -- command [args...]
op-exec op://vault/item  # print exports (for debugging)
```

## Examples

```bash
op-exec op://Development/my-app-config -- npm start
op-exec op://Work/aws-creds -- aws s3 ls
```

## Installation

```bash
brew install nsheaps/devsetup/op-exec
```

## Requirements

- [1Password CLI](https://developer.1password.com/docs/cli) (`op`)
- `jq`

## License

MIT
