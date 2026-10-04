# pmpx-plugin-yarn

> 中文版见 [README_CN.md](README_CN.md)

The **yarn** backend for [pmpx](https://crates.io/crates/pmpx). It maps pmpx's verbs onto yarn
commands, and does nothing else: no file reads, no environment, no network.

```console
$ pmpx install             # in a Node project → yarn install
$ pmpx install lodash      #                   → yarn add lodash
$ pmpx run dev --port 3000 #                   → yarn run dev --port 3000
```

## The mapping

| pmpx | yarn |
| ---- | ---- |
| `install` | `yarn install` |
| `install <pkg>...` | `yarn add <pkg>...` |
| `remove <pkg>...` | `yarn remove <pkg>...` |
| `run <script> [arg]...` | `yarn run <script> [arg]...` |
| `build [arg]...` | `yarn build [arg]...` |
| `test [arg]...` | `yarn test [arg]...` |
| `update` | `yarn upgrade` / `yarn up` |
| `update <pkg>...` | `yarn upgrade <pkg>...` / `yarn up <pkg>...` |
| `exec [arg]...` | `yarn exec [arg]...` |

**Not one of these inserts a `--`.** yarn hands what follows a script name to the script
unchanged, so an argument that looks like an option arrives intact:

```console
$ yarn run probe --foo
GOT:--foo
```

`yarn run probe -- --foo` also gets there -- yarn strips the `--` itself -- but there is
nothing to strip it for, so the plugin does not add one. This is the opposite of what a backend
for a tool that parses options after the script name has to do.

`install` with a package name is `yarn add`, not `yarn install <pkg>`: adding is a command of
its own, and it installs what it adds.

`exec` is supported: `yarn exec` resolves a binary out of the project's dependencies and runs
it.

## classic and berry

`update` is the one verb yarn spells differently in its two generations. Berry (yarn 2+) renamed
`upgrade` to `up` and deprecated the old spelling, so the command depends on which yarn the
project uses -- and both arities are affected.

The plugin does not read `.yarnrc.yml`, or anything else, to find out. pmpx's detect layer has
already looked, and reports which files it matched through `Context::matched`. The manifest
declares `.yarnrc.yml` as strong evidence, and the plugin only asks whether it was among them:

```rust
Verb::Update => CommandSpec::new("yarn")
    .arg(if ctx.has_matched(".yarnrc.yml") { "up" } else { "upgrade" })
    .args(args.iter()),
```

No `.yarnrc.yml` means classic, so `yarn upgrade`. Every other verb maps identically in both
generations.

## Install

```console
$ pmpx plugin add yarn
```

## Detection

From `pmpx-plugin.toml`, which travels with this crate:

| File | Weight | What it proves |
| ---- | ------ | -------------- |
| `yarn.lock` | strong (100) | the project was actually resolved by yarn |
| `.yarnrc.yml` | strong (100) | yarn berry, and which generation to map `update` for |
| `package.json` | weak (10) | the ecosystem, not the tool |

`package.json` is weak on purpose: npm, pnpm and yarn all sit in the same tree, so it says
nothing about which of them owns the project. The lockfile does. A tree with no lockfile scores
10 and can lose to another ecosystem -- that is what the pins in `.pmpx.toml` are for.

## Requirements

Rust **1.82+**, which is the contract crate's floor. The mapping itself needs nothing beyond the
2021 edition.

## Repository

<https://github.com/pmpx-rs/pmpx-plugin-yarn>

## License

MIT — see [LICENSE](LICENSE).
