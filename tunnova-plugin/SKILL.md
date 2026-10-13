---
name: tunnova-plugin
description: 为 Tunnova（隧道网关固件，主仓库 almightyYantao/tunnova-sdwan）编写、打包、签名、测试和发布插件（.tnp）。覆盖插件目录结构、manifest 规则、前端 window.Tunnova SDK、BusyBox sh 后端写法、uci tnp_<id> 配置、procd 服务、本地打包签名、上测试设备实测、经插件中心发布。当用户要给 Tunnova 加插件 / 扩展功能 / 写 .tnp、改现有插件、或问插件怎么写 / 怎么发布时使用。
---

# Tunnova 插件开发

插件让 Tunnova 不重刷固件就多出后台菜单页 + 后端脚本 / 服务。设计文档（以它为准，有冲突先读它）：`docs/plugin-architecture.md`。完整示例：`plugins/cron/`（定时任务，覆盖了所有环节，**写新插件先照着它抄**）。

仓库：主仓库 `tunnova-sdwan`（本地路径问用户或看当前工作目录）。对外文案、界面一律中文；**白标：任何地方不出现 ImmortalWrt / OpenWrt / LuCI 字样**（插件名、描述、界面、日志都算）。

## 1. 目录结构

```
plugins/<id>/
  manifest.json         必需
  ui/src/index.jsx      可选：前端页面（打包成 ui/ui.js）；ui/ 下其它静态文件原样带上
  backend               可选：后端脚本（BusyBox sh），前端经 exec 调用；ACL 只放行这个文件名
  init                  可选：procd / rc.common 服务脚本（固件生成包装脚本 /etc/init.d/tnp-<id> 引入它，procd 服务名是 tnp-<id>；脚本里别用 $0 / dirname 找自己的目录，写绝对路径）
  uci/tnp_<id>          可选：默认配置，安装时 /etc/config/tnp_<id> 不存在才拷
  migrate               可选：覆盖安装（升级插件）后执行一次，参数是旧版本号
  bin/x86_64/…  bin/aarch64/…   可选：二进制，安装时链接成 bin/current
```

设备上解包到 `/etc/tunnova/plugins/<id>/`，前端经软链 `/www/tunnova/plugins/<id>` → `ui/` 访问。
**升级 / 重装插件时这个目录会被整个换成新包**，运行中产生的数据不能放这里（见第 3 节）。

## 2. manifest.json

```json
{
  "id": "cron",                 // ^[a-z][a-z0-9]{1,23}$ —— 不能有 - 和 _
  "name": "定时任务",
  "version": "1.0.0",           // X.Y.Z 或 X.Y.Z-beta.N；升级必须变大（插件中心按它判断有没有更新）
  "sdk": 1,                     // 必须等于固件的 SDK 版本，目前是 1
  "min_tunnova": "v1.2.5",      // 用到的固件功能从哪个版本起有；插件机制本身从 v1.2.5 起
  "arch": ["all"],              // 或 ["x86_64","aarch64"]（带 bin/ 时按实际给）
  "description": "一句话说清楚做什么",
  "author": "Tunnova",          // 插件管理页和插件中心会显示
  "license": "plugin-cron"      // 可选：收费插件填功能 id（docs/licensing.md）。没有这个功能的授权时不能安装 / 更新，
                                //   插件中心显示「需要授权」；已装的照常运行。签发授权时用 -add plugin-cron 开通
}
```
- 插件页面里判断授权：`window.Tunnova.license.has('policy-route')`；没授权时 `window.Tunnova.license.showLocked('<功能 id>')` 弹统一提示。

## 3. 后端（backend / init / migrate）

- **BusyBox ash，不是 bash**：没有 `[[ ]]`、数组、`od`、`cat -A`、`pgrep -x`（判断服务用 `/etc/init.d/<svc> status`）、`curl telnet://`；`sleep` 不支持小数间隔。写完用 `sh -n` 检查语法。
- 读配置：`. /lib/functions.sh; config_load tnp_<id>; config_foreach fn <类型>`；输出 JSON：`. /usr/share/libubox/jshn.sh`（`json_init / json_add_* / json_dump`）。
- **管道里的 `while` 跑在子 shell**，在里面 `json_add_*` 会丢：先写临时文件再 `while … done < 文件`。
- awk 里先 `k=$2` 存好再清字段（`$1=$2=""` 之后再用 `$2` 当键就错了）。
- 子命令约定：`status`（只读，前端轮询用）/ `apply`（让配置生效）/ 其它动作。**`status` / `list` / `log` 这类只读子命令不会记操作审计**，改东西的子命令会自动记。
- 参数一律校验（`case "$x" in ''|*[!A-Za-z0-9_]*) exit 1;; esac`），不要把用户输入直接拼进 `sh -c`，除非这就是功能本身（如「执行命令」）并在界面写清楚以 root 运行。
- 写系统文件（crontab、dnsmasq 配置等）只动自己的**标记段**（`# tnp-<id> begin` … `# tnp-<id> end`），`init` 的 `stop` 里删掉标记段——停用 / 卸载会先 stop 再 disable。
- 日志：`logger -t tnp-<id> "…"`；要进「事件与告警」：`. /usr/lib/tunnova/lib.sh; event info|warn|error system "<对象>" "<内容>"`。
- 持久数据（下载的名单、记录、状态）放 **`/etc/tunnova/plugin-data/<id>/`**（插件目录外：升级插件、刷固件都保留；v1.3.6 起卸载插件时固件一起删），临时的放 `/tmp`。
- 同一时间只能跑一份的操作（下载、生成）加锁：BusyBox 的 `flock` 没有 `-w`，用 `flock -n` 重试循环。
- BusyBox 的坑：`sort -u -o 文件 文件` 会同时把结果打到屏幕（用 `sort -u a > b && mv b a`）；`tr` 按字节处理，不能替换成中文（拼中文用 awk）。
- `init` 模板：
  ```sh
  #!/bin/sh /etc/rc.common
  START=96
  boot() { start; }
  start() { /etc/tunnova/plugins/<id>/backend apply; }
  stop()  { /etc/tunnova/plugins/<id>/backend clear; }
  ```
  需要常驻进程时用 `USE_PROCD=1` + `procd_open_instance`，并 `procd_set_param respawn`。
- 权限：固件自带的公共 ACL 组已放行 `/etc/tunnova/plugins/*/backend` 的 exec 和 `tnp_*` 的 uci 读写，**插件不带 acl.json**。要读写别的配置 / 调别的 ubus 对象，直接用（root 会话本来就有）；要调别的脚本，从 backend 里调。

## 4. 前端（ui/src/index.jsx）

只从 `window.Tunnova` 拿东西；`react`、`antd` 由打包脚本转到主程序那份（**不能用 react-dom**）。图标从 `@ant-design/icons` 正常 import（只打进用到的）。

```jsx
import React, { useState } from 'react';
import { App, Card, Form, Input, Modal, Table } from 'antd';
import { ClockCircleOutlined } from '@ant-design/icons';

const { api, ui, data } = window.Tunnova;
const { PageHeader, EmptyBox, Switch, Button, useAct, useOutput } = ui;
const B = '/etc/tunnova/plugins/<id>/backend';

function Page({ arg, setCrumb }) {
  const [d, reload] = data.usePoll(async () => ({
    cfg: api.byType(await api.uciSafe('tnp_<id>'), '<类型>'),
    st: await api.execJSON(B, ['status']).catch(() => ({})),
  }), 10000);
  if (!d) return <Card loading />;
  return <PageHeader icon={<ClockCircleOutlined />} title="…" desc="…" />;
}

export default { pages: [{ id: 'main', title: '定时任务', icon: ClockCircleOutlined, component: Page }] };
```

- 页面 id 只能 `[a-z0-9-]`，路由变成 `#x-<插件id>-<页面id>`；`detail` 是详情页组件（`#x-…/<参数>` 时用，组件收到 `arg`）。
- **`group` 写对应的已有分组**（如「分流」「网络」「运维」），页面就挂进那个分组；不写进最底下的「扩展」。和功能相关的页面别都堆在「扩展」。
- SDK（sdk 1）：
  - `api`：`ubus, uci{get,set,add,del,commit,order}, uciSafe, byType, replaceSection, delSection, setClean, exec, execJSON, out, safeApply, audit, asList, lines, fmtBytes, fmtDur, fmtAgo, REMOTE`（v1.3.6 起还有 `fmtTime`；要兼容老固件就插件自己带一份）
  - 删除 uci 选项（清空列表）：`api.ubus('uci', 'delete', { config, section, option })`（`api.uci` 没有删选项的方法）
  - `data.usePoll(fn, 间隔ms)` → `[data, reload, loading]`；`reload()` 返回 Promise，保存后 `await reload()` 列表就是新的
  - `ui`：`PageHeader, Stats, EmptyBox, Mono, StateTag, useAct, useOutput, Switch, Button, SaveBar, useDirty, Chart`
- 写配置：`await api.replaceSection(cfg, 类型, 名字, 值)` 或 `api.uci.set` → **`await api.uci.commit(cfg)`**（commit 时自动记审计）→ 调 `backend apply`。
- 开关用 `ui.Switch`、按钮用 `ui.Button`：`onChange` / `onClick` 返回 Promise 就自动转圈、成功前保持新状态——**handler 一定要 return / async**，否则没有加载效果。
- `useAct()(异步函数, '成功提示')` 统一处理成功 / 报错提示；长输出用 `useOutput()(标题, () => exec(...))`。
- 远程模式（`api.REMOTE` 非空，经 Tunnova Cloud 打开）：不能上传文件；地址都用相对路径，别写死 `http://设备IP`。
- 页面崩了只会显示错误卡片，不影响整个后台；但还是要处理好空数据（`d.xxx || []`）。
- 文案：说用户能看懂的话（「重连隧道」而不是「ipsec-reconnect」），界面中文。
- **时间**统一显示成 `yyyy-MM-dd HH:mm:ss`（不用 `toLocaleString()`）。
- **表格**：单元格不换行（放不下横向滚动）、首列固定在左、「启用」开关放操作列前面并和操作列一起固定在右、操作列表头写「操作」、操作列左边加分割线。
  插件的表格是 antd 原生 Table，要自己做：模块里注入一次样式 `.tnp-nowrap .ant-table-cell{white-space:nowrap}`，表格加 `className="tnp-nowrap" scroll={{ x: 'max-content' }}`，列上写 `fixed`，操作列 `onCell` / `onHeaderCell` 加 `borderLeft`。
- **二次确认**：会断流、断隧道、删数据、装东西的按钮和开关都要先确认。
  - 不要在开关的 `onChange` 里直接 `return modal.confirm(...)`：它也是 thenable，开关会以为在切换，点「取消」也先显示成新状态 8 秒。
  - 写成「确定 resolve、取消 reject」的 Promise：
    ```js
    const ask = (modal, { onOk, ...o }) => new Promise((resolve, reject) => modal.confirm({ ...o,
      onOk: async () => { try { await onOk(); resolve(); } catch (e) { reject(e); throw e; } },
      onCancel: () => reject(new Error('已取消')) }));
    // <Switch onChange={v => v ? toggle(v) : ask(modal, { title: '停用…？', okText: '停用', onOk: () => toggle(v) })} />
    ```
  - v1.3.6 起 `ui.Switch` / `ui.Button` 支持 `confirm` 参数，但老固件不认，插件里按上面自己写。
- 插件往分流应用库里加应用：写 `managed '<插件 id>'`（可选 `managed_label` 标签文字），应用库显示「由插件维护」、只读、链到插件页面，「同步」不动它的数据。

## 5. 打包、签名

```sh
cd <主仓库目录>
plugins/build.sh plugins/<id>              # → plugins/out/<id>-<版本>.tnp（已签名）
plugins/build.sh plugins/<id> --unsigned   # 开发包：只能在开发模式下装
```
- 需要：`web/node_modules`（没有就 `cd web && npm ci`）、Docker（打包和 signify 签名在容器里做，Mac 的 tar 会带扩展属性）。
- 私钥 `plugins/keys/tunnova-plugin.sec`（不进仓库）；CI 用 secret `PLUGIN_SIGN_KEY`。公钥随固件 `files/usr/share/tunnova/plugin-keys/<指纹>`。

## 6. 在测试设备上测

用一台装好 Tunnova 的测试设备（真机或虚拟机），地址和登录方式问用户。

```sh
# 传包 → 安装 → 看状态
cat plugins/out/<id>-<版本>.tnp | <到测试设备> 'cat > /tmp/tunnova-up/plugin.tnp'
/usr/libexec/tunnova-plugin install /tmp/tunnova-up/plugin.tnp
/usr/libexec/tunnova-plugin list
/etc/tunnova/plugins/<id>/backend status
```
必测清单：
1. 安装 → `list` 状态 ok；`/www/tunnova/plugins/<id>`、`/etc/init.d/tnp-<id>`、`/etc/rc.d/S??tnp-<id>` 都在；默认配置拷过去了
2. 功能本身：改配置 → `backend apply` → 系统里真的生效（看生成的文件 / 进程 / 日志）
3. 停用 → 标记段、链接、服务都清掉；启用 → 恢复
4. 模拟固件升级：删掉 `/www/tunnova/plugins/<id>`、`/etc/init.d/tnp-<id>`、rc.d 链接、标记段 → `/etc/init.d/tunnova-plugins start` → 全部恢复
5. 版本 +1 重打包覆盖安装 → 配置保留（有 `migrate` 就验证迁移）
6. 卸载 → 目录、`/etc/config/tnp_<id>`、链接、标记段都没了
7. 前端：手动部署前端要先 `./stamp.sh web/dist`（否则浏览器用缓存的旧 app.js）；插件页本身随插件包走，`?v=` 自动带版本
8. 设备版本低于 `min_tunnova` 时会拒装——测试机用的是开发版就临时改 `/etc/tunnova_release`（**测完改回去**）

## 7. 发布到插件中心

插件源：`<插件源>/index.json` + `<插件源>/<id>/<id>-<版本>.tnp`。出厂默认 `https://r2.tunnova.com/tunnova/plugins`（公开读），也可以在设备「运维 → 插件管理 → 插件中心 → 插件源」单独设地址和私有桶密钥（`uci tunnova.plugins.url / access_key / secret_key / region`）。CI 发布的是默认位置（`<S3_PREFIX>/plugins/`）；单独设了插件源的设备，要把插件包和 index.json 传到那个地址（`plugins/index.py` 生成目录）。

- 改完插件、**版本号加一**，合进 main 后：打 tag `plugins-vYYYY.MM.DD`（或 Actions 里手动运行 `plugins` 工作流）。CI 打包全部插件、签名、生成目录，直接上传到升级源存储（插件包先、`index.json` 最后），合并旧目录、保留旧版本。
- 设备「运维 → 插件管理 → 插件中心」：未安装显示「安装」，有新版本显示「更新」，不兼容写明原因。

## 8. 交付前自查

- [ ] manifest：id 合法、版本号比上一版大、`min_tunnova` 对、`author` 填了
- [ ] 后端 `sh -n` 通过，没用 bash 语法；只读子命令叫 `status` / `list`
- [ ] 只改自己的标记段，`init stop` 能清干净
- [ ] 前端只用 `window.Tunnova`，handler 都返回 Promise，保存后 `await reload()`
- [ ] 没有 ImmortalWrt / OpenWrt / LuCI 字样
- [ ] 数据放 `/etc/tunnova/plugin-data/<id>/`；菜单 `group` 挂对分组
- [ ] 表格不换行、固定列、操作列表头；会改状态的操作有二次确认（取消后开关能退回）
- [ ] `plugins/build.sh` 打出签名包，在测试设备上把第 6 节清单跑完

## 更新日志和版本（2026-10-10 起）

- 每个插件目录要有 `CHANGELOG.json`（新的在前，每条 `version / date / changes`），`manifest.json` 的 `version` 必须有对应条目，否则 `plugins/build.sh` 拒绝打包。
- 改插件发新版本：manifest 版本号 +1 → CHANGELOG.json 最前面加一条 → 打包 / 发布。插件目录发布时会合并旧版本，设备可以在插件中心选版本（含降级）、看更新日志。
- 设备端：`tunnova-plugin changelog <id>`、`tunnova-plugin install-repo <id> [版本]`。
