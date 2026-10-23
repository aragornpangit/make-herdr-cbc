---
name: make-herdr-cbc
description: 把 CodeBuddy Code（cbc）做成 herdr 原生 agent kind，产出可用的 herdr-<version>-cbc 二进制。当 herdr 认不出 cbc、agent prompt 报 agent_not_ready，或需要给上游 master 加一个新 agent kind 时使用。触发词：/make-herdr-cbc、make-herdr-cbc、给 herdr 加 codebuddy、herdr-cbc。
---

# Make herdr-cbc：给 herdr 原生加 CodeBuddy kind

把上游 herdr 源码 clone 下来，在 `detect` 层加一个 `Agent::Codebuddy`，编译出独立二进制，不动系统里正在跑的 `herdr`。

## 产出物

| 项 | 位置 |
|---|---|
| 源码 | `~/pxy/tools/herdr/src`，分支 `feat/codebuddy-kind` |
| 二进制 | `~/.local/bin/herdr-<上游版本>-cbc`（例：herdr-0.9.5-cbc）|
| 补丁 | `git format-patch -1 -o ~/pxy/tools/herdr/` 得到的 `0001-*.patch` |
| 远端 | `mine` → https://github.com/aragornpangit/herdr-cbc （default branch `main`）|

## 为什么要打补丁

上游 `Agent` 枚举里没有 CodeBuddy，而 cbc 是 node 包裹启动（`node .../bin/cbc`），所以：

- pane 里跑着 cbc，`herdr agent list` 里没有条目；
- `herdr agent prompt` 报 `agent_not_ready` / "is no longer the pane foreground process"；
- 靠 hook 上报（`pane report-agent` + `custom:codebuddy:<pane>`）只能上报状态，prompt / send-keys 依旧被拒（hook shell 跑在独立 session/pgid，不在 pane 前台进程组）；
- 跨 pane 通信只能退回 `herdr pane send-text` + `send-keys enter`（或 `herdr-ml`）。

补丁落地后，进程识别与屏幕状态判定都由 herdr 自己完成，`agent list / prompt / start` 全部可用，也不需要 hook 集成。

## 前置

```bash
cargo --version          # 需与仓库 rust-toolchain.toml 一致（0.9.5 是 1.96.1）
command -v cbc           # CodeBuddy Code 已安装
herdr --version          # 记录上游版本，用于命名产物
```

## 步骤

### 1. 取源码

```bash
mkdir -p ~/pxy/tools/herdr && cd ~/pxy/tools/herdr
git clone --depth 1 https://github.com/herdrdev/herdr.git src
cd src && git log --oneline -1
```

浅克隆够编译，但**推送前必须 `git fetch --unshallow`**（见「坑 2」）。

### 2. 找齐改动点

先确认上游还没有 codebuddy，再以现成 agent（`muse` 最贴近）为模板：

```bash
grep -rin 'codebuddy\|cbc' . | head            # 期望无输出
grep -rn 'Muse' --include=*.rs src | sed 's/^/  /'   # 需要照抄的位置
```

固定 4 个文件：

| 文件 | 改什么 |
|---|---|
| `src/detect/mod.rs` | `Agent` 枚举加 `Codebuddy`；`ALL` 24→25；`SCREEN_MANIFEST_AGENTS` 22→23；`agent_label` = `"codebuddy"`；`interactive_agent_executable` = `"cbc"`；`lookup_agent` 加 `"codebuddy" \| "codebuddy-code" \| "cbc"`；node 包裹分支 `Some(Agent::Qwen \| Agent::Cline \| Agent::Letta \| Agent::Codebuddy)` |
| `src/detect/manifest.rs` | `BUNDLED_MANIFESTS` 加 `("codebuddy", include_str!("manifests/codebuddy.toml"))` |
| `src/detect/manifests/codebuddy.toml` | 新建屏幕判定规则（working / idle / blocked）|
| `src/config/sound.rs` | `Some(Agent::Codebuddy) => AgentSoundSetting::Default,` |

`ALL` / `SCREEN_MANIFEST_AGENTS` 是定长数组，**只加元素不改长度会编译不过**——这正是可靠的漏改检查。

### 3. 采真实屏幕写 manifest

不要凭想象写正则，先采屏：

```bash
herdr pane list                                              # 找到正在跑 cbc 的 pane
herdr pane read w1:p5 --source detection --format text       # working 态
herdr pane read w1:p6 --source detection --format text       # idle 态
```

三个状态各一条规则，priority 从高到低：blocked(980) → working(970) → idle(950)：

- **blocked**：工具权限确认框，`contains = ["Are you sure you want to do this?"]`，`region = "bottom_non_empty_lines(14)"`，`visible_blocker = true`
- **working**：composer 上方的实时行 `✺ Mixing… (140s · running REPL · …)`，用 spinner 字形集 `[✳✴✵✶✷✸✹✺●⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏]` + 括号里的耗时/关键字 `running` 双重匹配
- **idle**：两条横线之间的 composer，`region = "prompt_box_body"`，`line_regex = ['^\\s*>']`

manifest 头部：

```toml
id = "codebuddy"
version = "<YYYY.MM.DD.N>"
min_engine_version = 2          # 引擎当前是 3，填 2 兼容
updated_at = "<ISO8601>Z"
aliases = ["codebuddy-code", "cbc"]
```

### 4. 编译与自测

```bash
cd ~/pxy/tools/herdr/src
cargo check --offline
cargo test --offline --bin herdr detect::        # 注意是 --bin，不是 --lib
cargo build --offline --release --bin herdr
```

再补两条单测（照 `muse` 的位置写）：`identify_agent("cbc")`、`identify_agent("/home/user/bin/cbc")`、以及 label/executable 对照表。

### 5. 安装成独立名字

**不要覆盖 `~/.local/bin/herdr`**——那会动到当前正在用的 server 会话。

```bash
install -m 0755 target/release/herdr ~/.local/bin/herdr-$(herdr --version | awk '{print $2}')-cbc
~/.local/bin/herdr-0.9.5-cbc --version
```

### 6. 隔离验证（关键一步）

不能拿主 session 试（会和线上 server 抢 socket），起一个嵌套 session，并**关掉 hook 上报**证明是原生识别：

```bash
herdr pane split --current --direction right --cwd "$PWD" --no-focus      # 得到 w1:p8
herdr pane run w1:p8 "HERDR_ENV=0 herdr-0.9.5-cbc --session cbctest"
sleep 10
S=$HOME/.config/herdr/sessions/cbctest/herdr.sock
HERDR_SOCKET_PATH=$S herdr-0.9.5-cbc pane run w1:p1 "HERDR_ENV=0 cbc"
sleep 14
HERDR_SOCKET_PATH=$S herdr-0.9.5-cbc agent list | jq -c '.result.agents[] | {pane:.pane_id,agent,agent_status}'
# 期望 agent=codebuddy，agent_status=idle
HERDR_SOCKET_PATH=$S herdr-0.9.5-cbc agent prompt w1:p1 "只回复 PONG" --wait --timeout 60000
# 期望 agent_prompted
```

离线校验 manifest 也很快，不用起 session：

```bash
./target/release/herdr agent explain --file /tmp/cbc_idle.txt --agent codebuddy
```

收尾：

```bash
herdr-0.9.5-cbc session stop cbctest && herdr-0.9.5-cbc session delete cbctest
herdr pane close w1:p8
```

### 7. 提交与推送

```bash
git checkout -b feat/codebuddy-kind
git add src/detect/mod.rs src/detect/manifest.rs src/detect/manifests/codebuddy.toml src/config/sound.rs
git commit -m "feat(detect): recognize CodeBuddy Code (cbc) as a built-in agent"
git format-patch -1 -o ~/pxy/tools/herdr/

git fetch --unshallow                                   # 坑 2：不 unshallow 推不上去
git remote add mine https://github.com/aragornpangit/herdr-cbc.git
git push -u mine feat/codebuddy-kind:main
```

仓库不存在时用 API 建（token 在 `~/.git-credentials`，**只取值不打印**）：

```bash
TOKEN=$(awk -F'[:@]' '/github/ {print $3}' ~/.git-credentials | head -1)
curl -s -X POST -H "Authorization: token $TOKEN" -H "Accept: application/vnd.github+json" \
  -d '{"name":"herdr-cbc","private":false}' https://api.github.com/user/repos
```

## 坑

1. **TOML 里的正则反斜杠**：manifest 用单引号字面量 `'^\\s*>'`；写成 `'^\\\\s*>'` 就变成匹配字面反斜杠，规则永不命中。改完先 `agent explain --file` 验一遍。
2. **浅克隆推不上去**：`git push` 报 `did not receive expected object ...`。先 `git fetch --unshallow`（1869 个提交，几分钟），再推。
3. **嵌套 herdr 被拦**：`HERDR_ENV=1` 时 herdr 拒绝启动嵌套 server，验证时必须 `HERDR_ENV=0`；同时内层 cbc 也要 `HERDR_ENV=0` 才是"无 hook"条件。
4. **`cargo test --lib` 跑错 crate**：主 crate 是 bin，过滤词会落到 `ghostty_vt`。用 `--bin herdr`。
5. **shell 引号吃变量**：`awk -F"[:@]" "/github/ {print $3}"` 里 `$3` 被 bash 展开成空，必须单引号包 awk 程序。
6. **0.9.3 的假象**：旧版二进制里能 strings 出 `id = "codebuddy"` 的 manifest，但没有进程识别——`agent list` 仍为空，`agent prompt` 仍失败。只有 manifest 不够，必须同时改 `detect/mod.rs`。
7. **别覆盖主二进制**：换成 `herdr` 会让正在跑的 server 与 CLI 版本错配；产物一律独立命名。

## 验收清单

- [ ] `herdr-<ver>-cbc agent start --help` 的 kind 列表含 `codebuddy`
- [ ] `cargo test --bin herdr detect::` 全绿
- [ ] 嵌套 session 里 `HERDR_ENV=0 cbc` 能被识别成 `agent=codebuddy`，状态在 idle/working/blocked 三态间正确切换
- [ ] `agent prompt` 对两个 cbc pane 都返回 `agent_prompted`，且 pane 里看到回复
- [ ] 主 `~/.local/bin/herdr` 未被改动，线上 session 无感
- [ ] 补丁已导出、分支已推送
