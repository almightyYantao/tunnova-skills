---
name: tunnova-component
description: Tunnova（隧道网关固件，主仓库 almightyYantao/tunnova-sdwan）组件热更新（.tnc）的开发、打包、影响评估、界面展示、测试设备实测和发布。组件 = 固件自带、可单独更新的用户态部分（第一期是隧道组件 tunnel：tunnova-tunnel、updown、TCP 封装中转、strongSwan 路由配置）。当用户要发隧道补丁但不想发固件、做 / 改组件更新机制、写组件更新的界面、评估一次组件更新会不会影响现有隧道、或问组件怎么发布时使用。不负责插件（.tnp，走 tunnova-plugin）和整固件发版。
---

# Tunnova 组件热更新

设计文档（以它为准，有冲突先读它）：`docs/component-update.md`。插件机制（签名、分发、开机恢复都复用）：`docs/plugin-architecture.md`。

仓库：主仓库 `tunnova-sdwan`（本地路径问用户或看当前工作目录）。对外文案、界面一律中文；白标：任何地方不出现 ImmortalWrt / OpenWrt / LuCI。

> 已实现（2026-10-10）。代码位置：设备端 `files/usr/libexec/tunnova-component`（白名单 `files/usr/share/tunnova/components/<id>.paths`，
> 开机 `files/etc/init.d/tunnova-components`）、演算 `tunnova-tunnel vti-plan`、打包 `components/build.sh` / `components/index.py` / `.github/workflows/components.yml`、
> 设备后台 `web/src/pages/Components.jsx`（「运维 → 组件更新」单独一页）、平台 `cloud/server/components.go` + `cloud/web/src/pages/Components.jsx`。
> 改之前先读对应代码，命令以 `tunnova-component` 的用法说明为准。

## 铁律：更新前界面必须讲清对现有隧道的影响

这是本 skill 最重要的一条（设计文档 7.7）。写任何安装 / 更新 / 回滚组件的入口（设备后台、Cloud、命令行输出）时逐条检查：

1. **确认之前**就显示影响，数据来自设备上实时跑的 `tunnova-component plan`，不能只用包里的 `disrupt` 字段。
2. 第一行是一句话结论：「本次更新不会影响现有隧道」或「本次更新会让以下 N 条隧道短暂中断」。
3. 有影响时列出每条受影响隧道：名称、当前传输方式（看 `/var/run/tunnova-vti/<名称>.transport`，不是 uci 里配的）、影响方式（重新协商 / 重建网卡 / TCP 重连）、预计中断时长、经过它的分流规则数；不受影响的折叠成「其余 M 条不受影响」。
4. 列表上的标签：绿「不影响隧道」/ 橙「N 条隧道会短暂中断」/ 红「全部 IPsec 隧道会重新协商」。
5. 有中断时：默认选维护窗口；确认按钮写明后果（「更新并中断 2 条隧道」），不用「确定」。
6. 回滚同样先 plan 再确认；自动回滚后的提示写清版本、哪条检查没过、哪些隧道又中断了一次。
7. Cloud 批量推送：每台设备一列「影响」，可按影响筛选；确认框汇总「X 台不影响，Y 台共 Z 条隧道会短暂中断」。
8. plan 拿不到结果（离线、超时、校验失败）→ 不允许安装，显示原因。

交付前自查：在测试设备上至少各做一次「不影响隧道」和「有中断」的更新，截图确认弹窗内容和上面一致。

## 什么动作会影响隧道（判断依据）

挂载（bind mount）文件本身不碰内核 SA 和 VTI 网卡。影响来自让新文件生效的动作，**只对内容真的变了（sha256 不同）的文件执行**：

| 变了什么 | 动作 | 影响 |
|---|---|---|
| 脚本逻辑变了、生成的隧道配置没变 | 无 | 不中断 |
| 生成的 strongSwan 连接配置变了 | `ipsec-reload` | 只有配置变了的连接重新协商，几秒到几十秒 |
| VTI 网卡参数变了（`tunnova-tunnel` 里 `vti_one` 的 `want` 字符串变化） | `ipsec-reload`（重建网卡） | 那几条断几秒；分流路由由 `tunnova-route` 下一个探测周期（`tunnova.global.interval`，默认 5 秒）自动补回 |
| TCP 封装中转换了 | `restart:tunnova-tcpencap` | 所有**实际**走 TCP 的隧道断开重连，实测约 10–20 秒；走 UDP 的不受影响 |
| `/etc/strongswan.d/` 全局配置变了 | `ipsec-reload` | 按 `all` 处理：全部 IPsec 重新协商 |

判断「网卡会不会重建」：用新脚本生成一份 `$VTI_RUN/<名称>` 的 `want` 值和现在的比较（`gw lip key ip eff [rip]`）。2026-10-10 的修复在 TCP 隧道的 `want` 里加了回环地址，所以所有 TCP 隧道的网卡都会重建一次——这类「改了 want 格式」的改动一定要在 notes 里写出来。

## 做一个组件更新的流程

1. **确认能走组件**：改动只在 `/usr/share/tunnova/components/<id>.paths` 白名单里的文件；动了内核、strongSwan 二进制 / C 插件、前端 `app.js`、`tunnova-agent` 主程序 → 不能走组件，要发固件。状态 JSON 只加字段，不改不删（老固件的网页要能继续显示）。
2. **代码照常改、照常合进 main**（修复要进下一个固件版本）。
3. **打包**：`components/build.sh <id>`，版本 `<固件版本>-<序号>`（如 `1.3.4-1`），`firmware` 写这个补丁能用的固件版本（可以多个）。**选序号前先 `git tag -l 'comp-<id>-*'` 看有没有被占用**（可能别的会话已经发过同号）；`build.sh` 按 `actions` 算 `disrupt`，只能手动调高。
4. **写 notes**（会显示在确认框里）：修了什么 + 对隧道的影响，用用户看得懂的话，比如「修复两条隧道同时走 TCP 封装时第二条不通。走 TCP 封装的隧道会重连一次（约 10–20 秒），走 UDP 的不受影响。」
5. **测试设备实测**（地址和登录方式问用户；设备是 BusyBox：没有 nproc / paste / diff，sleep 不支持小数，nc 不支持 -z）：
   - 装之前记基线：`swanctl --list-sas` 每条的 `established Ns ago`、`tunnova` 出口状态、`/var/run/tunnova-vti/*.transport`；
   - 先跑 `plan`，记下它预测的受影响隧道；
   - 安装后对比：预测「不受影响」的隧道 SA 建立时间不能重置；受影响的实际中断时长和预测对得上（隧道内持续 `ping` 对端地址看丢了多久）；
   - 试用期通过变 `active`；再故意装一个坏包（比如脚本语法错误），确认自动回滚、隧道恢复；
   - 测速用隧道对端网段里的测速服务（地址问用户），`curl --interface ipsec_<名称>` 绑网卡。
6. **发布**：打 tag `comp-<id>-<版本>`，CI 产出 `.tnc` 和 `components/index.json`；Cloud 上先推试点设备，看影响列和回报，再推其余。
   - **iwan 组件带了新的 iwansd**（`iwansd/iwansd.c` 的 `VERSION` 变了）：独立二进制要同版本发布，tag `iwansd-v<VERSION>`（`iwansd.yml` 出 linux amd64 / arm64、macOS 客户端、install.sh）。改动合进 main 时 CI 发现这个版本还没有 tag 会自动发布并打 tag；发版汇报里核对升级源 `iwansd/latest/VERSION` 已是新版本。
7. **随固件自动同步**：每发一个正式版固件 vX.Y.Z，CI（`components-sync.yml` → `components/sync.sh`）自动给每个组件打 `X.Y.Z-0`（带脚本、不带中转二进制），适用 `compat_from`（`component.json`，默认 v1.3.5）到这一版。这一版固件自己装 `-0` 没有变化（设备显示「已是最新」），更早的固件装它不用刷机。改动依赖组件之外的东西时把 `compat_from` 调到这一版。

## 现有组件（白名单 files/usr/share/tunnova/components/<id>.paths）

- `tunnel` 隧道、`route` 分流（respawn，不能 restart）、`mon` 监控（含指标 / 采集器）、`iwan` iWAN（iwansd 换了：直接用 iwansd 起的本机服务端实例会被重新拉起、接入的客户端全部断开重连；经 iwan-run 起的客户端隧道不重启，要 `tunnova-iwan reconnect <隧道>` 才换上新程序。`etc/init.d/iwan` 是 `reload:iwan`，装完自动 reload，启动参数变了的实例会重启）、`sys` 系统工具（按需调用）。
- 不做成组件：tunnova-ctl（救命通道）、cloud / agent、登录 / 账号 / 管理面安全 / 只读、授权、插件和组件机制本身、lib.sh。理由见设计文档 3.1。
- 加新组件：写 `<id>.paths`（随固件发）→ 想清楚动作（能 respawn 就别 restart，restart 会先 stop）→ 在 `impact` / `effects` 里补这个组件的影响 → 健康检查（`snapshot` / `check`）→ 实测预测和实际对得上。

## 更新日志和版本

- 发组件前先在 `components/<id>/CHANGELOG.json` 最前面加一条（`version / date / firmware / changes / impact`），`build.sh` 没有这条就拒绝打包。
- `changes` 写用户看得懂的修复内容，`impact` 写对隧道 / 流量的影响（界面「更新日志」和确认框都会显示）。
- 修复照常合进主干；新固件构建会把 CHANGELOG 带进固件，设备显示「固件自带，已包含到 x」。
- 设备可以选版本：`plan repo:<id>@<版本>`、`install <id> --version <版本>`；平台「组件更新」页也能选版本。

## 踩过的坑

- **BusyBox sort**：不认 `-k2,2nr` 里的 r（倒序不生效），先 awk 算排序键再整体 `sort -r`（见 `vsort`）。
- **jshn 和管道**：`xxx | while read; do json_add_...; done` 在子 shell 里跑，加的 JSON 全丢；先写临时文件再 `while … done < 文件`。
- **BusyBox 工具**：`flock` 没有 `-w`（用 `flock -n` 循环重试）；jshn 的 `json_get_keys` 会改写带 `/` `-` 的键名（用 jsonfilter 按原路径查）；`nc` 没有 `-z`；`sleep` 不支持小数；没有 `diff` / `paste`。
- **restart 的副作用**：rc.common 的 restart 会把停用的服务拉起来（所以只 restart 在跑的）；分流的 restart 会先 stop 清掉整张表（所以用 respawn）。
- **bind mount**：目标是软链会跟随过去（所以 `/usr/sbin/tunnova-tcpencap` 是包装脚本）；运行中的程序占着文件时普通 `umount` 会 busy，用 `umount -l`。
- **中转二进制**：每次编译哈希不同，带上就一定重启中转、TCP 隧道全部重连；没改中转就别在 `include` 里写它。
- **测试包**：`SPEC=<规格> SRC=<目录> components/build.sh tunnel`；设备的固件版本号是 `/etc/tunnova_release` 里的，`firmware` 要和它一致。
- **已装组件的文件是 bind mount 上去的**：直接 `scp` / `cp` 覆盖 `/usr/libexec/tunnova-*`，写进的是**已装组件的存放目录**（`/etc/tunnova/components/<id>/files/…`），等于篡改了那个版本；`cp` 还会报「File exists」。临时试脚本前先 `mount | grep <文件名>`：挂着组件就别直接覆盖，改用测试包安装；改坏了用测试前的备份写回，并核对 sha256 和 `/etc/tunnova/components/<id>/manifest.json` 一致。
- **界面截图**：本机直连不到设备后台时，用本地代理把 `/ubus`、`/cgi-bin/tunnova-login` 经跳板机转给设备的 `curl http://127.0.0.1/...`，前端用本地 `web/dist`，puppeteer-core 驱动本机 Chrome；登录页用户名框默认填着 `admin`，先清空再输入；测试结束删临时账号、恢复改过的 uci。
- **确认框标题**：`Components.jsx` 的确认框里别再起叫 `verb` 的局部变量，会盖住上面的 `verb()` 函数（v1.3.7 前点「更新到」白屏就是这个）。

## 回复用户时

- 说清楚这次更新**会不会断隧道、断哪几条、多久**，以及有没有选维护窗口；
- 测试结果给实测数据（中断秒数、SA 建立时间对比），没测过的明确说没测；
- 发生过自动回滚要说出来，带原因。
