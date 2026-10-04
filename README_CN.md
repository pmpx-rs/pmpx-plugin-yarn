# pmpx-plugin-yarn

> English see [README.md](README.md)

[pmpx](https://crates.io/crates/pmpx) 的 **yarn** 后端。它只做一件事：把 pmpx 的动词映射成
yarn 命令 —— 不读文件、不看环境变量、不联网。

```console
$ pmpx install             # Node 项目里 → yarn install
$ pmpx install lodash      #              → yarn add lodash
$ pmpx run dev --port 3000 #              → yarn run dev --port 3000
```

## 映射表

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

**这里面没有一条要插 `--`。** yarn 会把脚本名后面的东西原样交给脚本，所以长得像选项的参数也能
完好到达：

```console
$ yarn run probe --foo
GOT:--foo
```

`yarn run probe -- --foo` 同样能到 —— `--` 是 yarn 自己剥掉的 —— 但既然没有东西需要剥，插件就
不插。这一点和"脚本名之后还要自己解析选项"的那类工具正好相反。

`install` 带包名时是 `yarn add`，不是 `yarn install <pkg>`：添加是它自己的一个命令，并且它会
把添加的东西装上。

`exec` 是支持的：`yarn exec` 会从项目依赖里解析出一个可执行文件并运行。

## classic 与 berry

`update` 是 yarn 两代之间唯一拼法不同的动词。berry（yarn 2+）把 `upgrade` 改名成 `up`，并弃用
了旧拼法，所以命令取决于项目用的是哪一代 —— 两种参数形态都受影响。

插件**不读** `.yarnrc.yml`，也不读别的东西来判断。pmpx 的检测层已经看过了，并通过
`Context::matched` 告诉插件命中了哪些文件。manifest 把 `.yarnrc.yml` 声明为强证据，插件只问它
在不在里面：

```rust
Verb::Update => CommandSpec::new("yarn")
    .arg(if ctx.has_matched(".yarnrc.yml") { "up" } else { "upgrade" })
    .args(args.iter()),
```

没有 `.yarnrc.yml` 就是 classic，即 `yarn upgrade`。其余动词在两代里映射完全一样。

## 安装

```console
$ pmpx plugin add yarn
```

## 检测

依据随这个 crate 一起发布的 `pmpx-plugin.toml`：

| 文件 | 权重 | 能证明什么 |
| ---- | ---- | ---------- |
| `yarn.lock` | 强（100） | 这个项目确实被 yarn 解析过 |
| `.yarnrc.yml` | 强（100） | 是 yarn berry，以及 `update` 该按哪一代映射 |
| `package.json` | 弱（10） | 只证明属于这个生态，不证明用了哪个工具 |

`package.json` 弱是刻意的：npm、pnpm、yarn 都待在同一棵树里，它说明不了这个项目归谁。锁文件才
能说明。一棵没有锁文件的树只有 10 分，可能输给别的生态 —— 这正是 `.pmpx.toml` 里那些固化项存在
的理由。

## 环境要求

Rust **1.82+**，这是契约 crate 的地板。映射本身只需要 2021 edition。

## 仓库

<https://github.com/pmpx-rs/pmpx-plugin-yarn>

## 许可

MIT —— 见 [LICENSE](LICENSE)。
