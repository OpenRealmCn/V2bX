# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

本仓库是 V2bX（基于多内核的 V2board 节点服务端，改自 XrayR）的 fork，origin 为 `OpenRealmCn/v2bx`（canonical 名 `OpenRealmCn/V2bX`）。

> 本目录常与另外两个仓库并排放在工作区 `D:\poject\OpenRealmWork\v2bx` 下（`v2board/` 面板、`V2bX-script/` 安装脚本），三者**各自独立带 `.git`**。上下文中若同时存在这三者，工作区根还有一份 CLAUDE.md 记录跨仓库契约。本文件自包含，单独 clone 本仓库时也能用。

## 分支与远端

- **默认且唯一的分支是 `dev_new`**。没有 `master`、也没有 `main`。
- 远端 URL 用 `OpenRealmCn/v2bx` 会 301 重定向到 `OpenRealmCn/V2bX`（canonical 大小写）。
- `release.yml` 的 push 触发器只认 `master` 和 `dev_new` 两个分支名。

## 构建

没有 Makefile。CI、Dockerfile、README 用的是同一条命令：

```bash
GOEXPERIMENT=jsonv2 go build -v -o build_assets/V2bX \
  -tags "sing xray hysteria2 with_quic with_grpc with_utls with_wireguard with_acme with_gvisor" \
  -trimpath -ldflags "-X 'github.com/InazumaV/V2bX/cmd.version=$version' -s -w -buildid="

go test ./...                                                                 # 全部测试
GOEXPERIMENT=jsonv2 go test ./conf -run TestXxx -tags "xray sing hysteria2"   # 单个测试
```

两个必须知道的点：

1. **`GOEXPERIMENT=jsonv2` 是强制的**，不是可选优化。`conf/conf.go`、`core/xray/xray.go`、`api/panel/user.go` 直接 import 了 `encoding/json/v2`，不加就编译不过。

2. **build tags 分两类**。`xray` / `sing` / `hysteria2` 是本项目自己的，每个只 gate `core/imports/` 下一个文件（该文件 blank-import 对应内核实现以触发注册）。`with_quic` / `with_grpc` / `with_utls` / `with_wireguard` / `with_acme` / `with_gvisor` 在本仓库源码里**根本不出现**——它们由 `go.mod` 的 `replace` 指向的 fork 消费。条件编译意味着「只编需要的内核」，减 tag 会真的删掉功能。

`CGO_ENABLED=0`；mips/mipsle 额外需要 `GOMIPS=softfloat`。

### ⚠️ Windows 上无法本地构建

本机（Windows）执行 `GOEXPERIMENT=jsonv2 go build` 会让 **Go 工具链自身 panic**：

```
panic: runtime error: invalid memory address or nil pointer dereference
goroutine 1 [running]:
cmd/go/internal/fips140.Init()
	cmd/go/internal/fips140/fips140.go:117 +0x101
```

已用一个无关的 hello module 做对照，**同样 panic**，证明这是 Go 工具链的预存问题、与本项目代码无关（`GOFIPS140=off` 也绕不过）。不带 `GOEXPERIMENT` 时构建能走到「build constraints exclude all Go files in .../encoding/json/v2」，即工具链本身正常。

**推论：在 Windows 上无法用本地编译验证改动。** 验证依赖解析用 `go mod verify` / `go list -m all`；验证真实编译只能推上去让 CI 跑（见「发布 Release」）。

## 已迁移到 OpenRealmCn 的依赖

所有指向上游 `wyx2685/` 的依赖与 URL **已全量切换**到自己的 fork。以下是实测得出的约束，**勿凭直觉改回**。

| 用途 | 仓库 | 默认分支 | 引用方式 |
|---|---|---|---|
| sing-box 内核 fork | `OpenRealmCn/sing-box_mod` | `dev-next` | `go.mod` 的 `replace` |
| Xray 内核 fork | `OpenRealmCn/Xray-core` | `main` | `go.mod` 的 `replace` |
| sing-vmess | `OpenRealmCn/sing-vmess` | `dev` | `go.mod` 的 `require` + `replace` |

### Go module 路径的两个硬约束

1. **`Xray-core` 的 `X` 必须大写。** Go module path 大小写敏感，canonical 名是 `OpenRealmCn/Xray-core`。写成小写 `xray-core` 会导致 sumdb 查询 404、构建失败（实测确认）。

2. **`sing-vmess` 的 `require` 行必须保留 `wyx2685`，靠 `replace` 重定向。** 因为该仓库的 `go.mod` 内部声明 `module github.com/wyx2685/sing-vmess`（module path 与 owner 名绑定）。直接把 `require` 改成 `OpenRealmCn` 会报：

   ```
   module declares its path as: github.com/wyx2685/sing-vmess
           but was required as: github.com/OpenRealmCn/sing-vmess
   ```

   这是**全仓库唯一一个刻意保留 `wyx2685` 字样的地方**，`go.mod:337-340` 有注释说明，不是遗漏。

### module path 本身刻意不改

`go.mod` 第一行是 `module github.com/InazumaV/V2bX`（第三个上游），**刻意保持不动**——改掉需要动全仓库每个 `.go` 文件的 import，以及 `release.yml` / `Dockerfile` 里硬编码的 ldflags 注入路径，且会让与上游的 diff 无法维护。同理 `cmd/*.go` 里的 `github.com/InazumaV/V2bX/...` import 也都保留。

注意区分：`cmd/install_linux.go` 里曾有一个指向 `InazumaV/V2bX-script` 的**脚本 URL**，那是要改的（已改为 `OpenRealmCn/V2bX-script`）；而 Go **import 路径**不改。

### go.sum 必须重算

fork 的 `h1:` 哈希与上游**不同**（内容相同但 module path 变了）。改动任何 `replace`/`require` 后必须跑：

```bash
GONOSUMDB='github.com/OpenRealmCn/*' go mod tidy
go mod verify                        # 应输出 all modules verified
go list -m all | grep OpenRealmCn    # 确认三个 replace 都指向 OpenRealmCn
```

`GONOSUMDB` 只在 sumdb 尚未索引新建仓库时需要（会返回 `404 not found: temporarily unavailable`，新建仓库的瞬态问题）。**索引完成后即可去掉**——已验证在 `GOPROXY=proxy.golang.org` + 默认 `GOSUMDB` 的 CI 等效环境下无需任何绕过即可解析。

### 域名拼写陷阱

上游脚本里有 `raw.githubusercontents.com`（多了个 "s"）——**这是个无效域名**，curl 返回 000、DNS 不解析。正确写法是 `raw.githubusercontent.com`。`cmd/install_linux.go` 的自更新命令在迁移前就是坏的，现已修正。写 URL 时注意别复制到这个错误拼写。

**例外（不要动）**：`example/geosite.dat` 里含 `githubusercontents.com` 字符串，那是**二进制 protobuf 地理路由数据库中的一条域名记录**（属于 GitHub 镜像域名列表，位于 `CN` 分类），**不是 URL**。该格式是长度前缀编码，手工替换会留下错误的长度前缀、损坏该记录及其后所有分类。校验脚本 URL 时要排除它：

```bash
grep -rn "githubusercontents" --exclude-dir=.git --exclude=CLAUDE.md --exclude="*.dat" .   # 应为空
```

## 发布 Release：必须用 workflow_dispatch 手动触发

**本仓库是 fork，上游「发布 Release 触发构建」的流程在这里不生效。** 实测确认：

- 创建并 Publish Release（`release: types: [published]`）**不产生任何 run**。
- 用 API 把 Release 转 draft 再转回 published，同样不触发。
- 调 `PUT /actions/workflows/{id}/enable` 返回 204、workflow 状态显示 `active`，但自动派发仍不生效。
- push（即使改了 `go.mod`/`*.go` 等 paths 过滤内的文件）也不触发——fork 的首次 push 只注册 workflow 文件。
- fork 的自动触发只能在 Web UI 的 Actions 页点确认解锁，**API 无法替代**。
- **但 `workflow_dispatch` 手动触发可以正常跑**（23 个 job 全绿）。

因此 `release.yml` 已改造（commit `5236b59`）：

- `workflow_dispatch` 新增可选输入 `release_tag`
- 合并出一个 `Resolve version` 步骤，按事件类型三分支（`release` 用 tag 名 / `dispatch` + tag 用输入值 / 其余回退 `github.sha`）
- 上传步骤的守卫从 `if: github.event_name == 'release'` 改为 `if: env.release_tag != ''`，`tag` 显式传 `env.release_tag`（fork 里 `github.ref` 是 `refs/heads/dev_new` 而非 tag，不能直接用），并加了 `overwrite: true`

**发布命令：**

```bash
# 1. 先确认代码已推到远端 —— Release 从远端 dev_new 构建，不是从本地工作树
git push origin dev_new

# 2. 触发构建并上传到指定 tag（tag 需已存在；23 个 job / 约 10 分钟）
gh workflow run release.yml --repo OpenRealmCn/V2bX --ref dev_new -f release_tag=v0.4.0-or1

# 不带 release_tag 则只产 Artifacts、不上传（与上游行为一致）
gh workflow run release.yml --repo OpenRealmCn/V2bX --ref dev_new

# 3. 观察进度
gh run list --repo OpenRealmCn/V2bX --limit 3
gh run view <run-id> --repo OpenRealmCn/V2bX --log | grep "resolved version"
```

**版本戳**：只有 `release_tag` 分支会把 tag 名注入 `cmd.version`，`V2bX version` 才输出可读版本号（如 `v0.4.0-or1`）；push 或无 tag 的 dispatch 会回退成 commit SHA。**所以发布一定要带 `release_tag`。**

**为什么不要用本地上传资产代替**：实测本地上行仅约 120 KB/s，单个 35 MB 的 zip 传了 10 分钟；而且本地产物版本戳是 SHA 而非 tag。让 runner 构建更快也更可靠。

产物名由 `.github/build/friendly-filenames.json` 的 `friendlyName` 决定，如 `linux-amd64` → `V2bX-linux-64.zip`。这三个是 `V2bX-script/install.sh` 按 `arch` 拼名所需的：`V2bX-linux-64.zip` / `V2bX-linux-arm64-v8a.zip` / `V2bX-linux-s390x.zip`（命名已核对一致）。

## 架构

`main.go` → `cmd.Run()`（cobra）。子命令：`server`（主入口）、`version`、`synctime`、`x25519`，以及 Linux 专有的 `start/stop/restart/log/update/uninstall`（`cmd/action_linux.go`、`cmd/install_linux.go`）。`cmd/cmd.go` blank-import `core/imports` 来激活已注册内核。

**配置是 JSON5 而不是 YAML**（允许注释，经 `common/json5` 剥离）。默认路径 `/etc/V2bX/config.json`（`cmd/server.go`），`--config/-c` 可覆盖。顶层结构 `{Log, Cores[], Nodes[]}`，`Nodes[].Include` 支持从文件或 URL 拉取节点配置。**viper 只用于 hysteria2 的子配置**（`core/hy2/node.go`），主配置完全不走 viper。参考样例见 `example/config.json`。

分层：

- **`core/`** — 内核抽象。`Core` 接口定义在 `core/interface.go`（`Start` / `Close` / `AddNode` / `DelNode` / `AddUsers` / `DelUsers` / `GetUserTrafficSlice` / `Protocols` / `Type`），注册表是 `core/core.go` 的 `cores map[string]factory` + `RegisterCore()`。三个实现在各自 `init()` 里自注册：`core/xray/xray.go`（`"xray"`）、`core/sing/sing.go`（`"sing"`）、`core/hy2/hy2.go`（`"hysteria2"`）。配置了多个内核时 `core/selector.go` 的 `Selector` 按 `Options.CoreName`/`Core` 或协议支持度把每个节点路由到某个内核，并用 `sync.Map` 记 tag→core。
- **`node/`** — 每个面板节点起一个 `Controller`（`node/controller.go`）：经 `api/panel.Client` 拉 NodeInfo/UserList、注册 limiter、取证书（`node/cert.go`、`node/lego.go`，lego ACME，模式 none/file/dns/http/self）、调 `core.AddNode/AddUsers`，再用 `common/task` 跑周期任务（节点信息监控、流量上报、在线 IP 上报、证书续签、动态限速）。
- **`limiter/`** — 以 tag 为键的全局注册表（`limiter/limiter.go`）：设备/IP 限制、限速令牌桶（juju/ratelimit）、域名与协议审计规则（`rule.go`）、动态限速（`dynamic.go`）。它**在内核内部通过 hook 生效**：xray 走 `core/xray/app/dispatcher/default.go`（一份 vendored 的 dispatcher 替换实现），sing 走 `core/sing/hook.go`，hy2 走 `core/hy2/hook.go`。
- **`api/panel/`** — resty 实现的 V2board 客户端。`Client` 是**具体结构体，不是接口**（尽管注释这么写），方法有 `GetNodeInfo` / `GetUserList` / `GetUserAlive` / `ReportUserTraffic` / `ReportNodeOnlineUsers`。用户列表用 msgpack 传输。调用的是 `/api/v1/server/UniProxy/{config,user,alivelist,push,alive}`，对端在 v2board 仓库的 `app/Http/Controllers/V1/Server/UniProxyController.php`。
- **`common/`** — `counter`（流量）、`rate`、`json5`、`task`、`systime`、`crypt`、`format`（`UserTag(tag, uuid)` 生成键）、`file`。

**节点 tag 是贯穿全局的主键**：格式 `[APIHost]-Type:NodeID`，limiter、core、controller 三方靠它对账。一个面板节点 = 一个 tag = 一个 limiter = 内核里的一个 inbound；而内核实例是跨节点共享的（单实例多节点，这是 V2bX 相对 XrayR 的核心改动）。

**热重载**：`conf/watch.go` 用 fsnotify 监听配置文件，变更时整体拆掉 nodes + core 再重建（`cmd/server.go` 里的 `serverHandle` 闭包），同时监听 `XRAY_DNS_PATH` / `SING_DNS_PATH` 环境变量指定的 DNS 文件。重启后会调 `runtime.GC()`——这个项目对内存占用敏感。

## 容易踩的坑

- `core/xray/app/dispatcher` 和 `core/xray/distro/all` 是 xray-core 包的**本地副本/定制版**。要改 xray 行为，通常是改这两个目录，而不是改 go.mod 依赖。`core/xray/app/dispatcher/config.pb.go` 是生成的 protobuf（对应 `config.proto`）。
- `conf.Options.UnmarshalJSON` 会**推迟内核相关字段的解析**：`Core` 未知时把原始内容留在 `RawOptions`，等 `Selector.AddNode` 选定内核后再二次 unmarshal。改配置结构体时顺序很关键。
- `.gitmodules` 声明了 `sing-box_mod` 子模块（已指向 `OpenRealmCn/sing-box_mod.git`），但**工作树里没有该目录**——fork 是通过 go.mod 的 `replace` 引入的，别指望子模块。
- 版本号靠 ldflags 注入 `cmd.version`（`cmd/version.go`，默认 `"TempVersion"`）。
- `cmd/install_linux.go` 的 `update` 子命令用 `bash <(curl -Ls https://raw.githubusercontent.com/OpenRealmCn/V2bX-script/master/install.sh)` 自更新——依赖 `V2bX-script` 仓库可达。

## 同步上游

```bash
git remote add upstream https://github.com/wyx2685/V2bX.git   # 只需一次
git fetch upstream dev_new
git merge upstream/dev_new      # 或 rebase
```

**合并后必然要重新检查这几处**（上游会把它们改回去）：

- `go.mod`：三条 `replace`/`require` 是否又变回 `wyx2685`。注意 `Xray-core` 大写、`sing-vmess` 保留 require+replace 这两个约束。改完跑 `go mod tidy` + `go mod verify`。
- `cmd/install_linux.go`：脚本 URL 是否又变回 `raw.githubusercontents.com/InazumaV/...`。
- `README.md` / `.gitmodules`：owner 名是否被改回。
- `.github/workflows/release.yml`：**上游会覆盖掉手动发布的改造**（`release_tag` 输入、`Resolve version` 步骤、`env.release_tag != ''` 守卫）。这三处是 fork 能发 Release 的前提，合并后必须确认还在。

一键自检（本目录下跑，应全部为空）：

```bash
# 例外：go.mod 的 sing-vmess require+replace 刻意保留 wyx2685（含其上方 3 行解释注释）
grep -rn "wyx2685" --exclude-dir=.git --exclude=CLAUDE.md . | grep -v "sing-vmess" | grep -v "go\.mod:[0-9]*://"
grep -rn "githubusercontents" --exclude-dir=.git --exclude=CLAUDE.md --exclude="*.dat" .   # 应为空
go list -m all | grep OpenRealmCn && go mod verify                       # 应输出 all modules verified
```

上游若新增了依赖 fork（例如新的 `replace`），需要先 `gh repo fork <upstream> --org OpenRealmCn --clone=false` 再切换，不要直接指向上游。