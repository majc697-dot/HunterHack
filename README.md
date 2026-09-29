# MT8666 车机永久 Root 方案技术总结（hh_rootd）

> 设备：车载主机
> 平台：MTK MT8666（内核报告 mt6771），Android 9 / API 28，userdebug 版本
> 分辨率 1920x720，A/B 分区（当前槽位 _a），无 vbmeta 分区
> 日期：2026-09-25
> 目标：让第三方应用 HunterHack（com.example.superstart）在重启后依然能执行 root 操作，
> 并在开机时自动注入方向盘媒体键桥接。

---

## 一、结论先行

最终落地的不是 Magisk，而是一套 **「init 托管的本机 socket root 守护 + SELinux 精准放通」** 方案：

```
init (root, 开机)
   └─ service hh_rootd  ──域转换──▶  u:r:su:s0（permissive 域）
          └─ while true; do toybox nc -s 127.0.0.1 -p 9753 -l /system/bin/sh; done
                      ▲
                      │ TCP（仅回环）
                      │
   HunterHack 应用（untrusted_app）
          └─ RootBridge.java：Socket 发命令，root 身份执行
                 ├─ 落盘并启动 netease_hk_v4.sh（方向盘键桥接）
                 ├─ pm disable 酷狗抢控组件
                 └─ 写 Settings.Global mobile_data=0（流量守护）
```

重启后全链路自动恢复，已实测验证（见第八节）。

---

## 二、方案演进与各方案的死因

### 路线 1：Magisk 修补 boot.img —— 放弃（会变砖）

- 设备 boot.img 带 **AVB1 传统签名**（镜像尾部 1322 字节 ASN.1 DER 结构）。
- 用 Python cryptography 库解析签名证书后发现：证书主体是 Android 主题 DN，
  但**签发者是 Mediatek（2014 年有效期），不是公开的 AOSP testkey**。
- 用公开 testkey.pk8（2008 版）验签失败 → 没有私钥就无法重签修补后的 boot。
- A/B 设备强刷未签名 boot 会导致启动校验不过，风险不可接受，故放弃。

### 路线 2：修补设备自带 su 二进制 —— 应用侧死路

设备自带一个 AOSP 简化版 su（仅允许 uid 0/2000）。用 capstone 反汇编后打了两处补丁：

1. uid 检查处（偏移 0xe40，cbz → b，字节 `08 00 00 14`）；
2. setgroups/setgid/setuid 三处调用 bl → `mov w0,#0`（字节 `00 00 80 52`，偏移 0xec4/0xeec/0xf10）。

shell/adb 下补丁版 su 可用，但**应用进程内彻底无效**，根因有二（`/proc/PID/status` 实测）：

- `NoNewPrivs: 1`：zygote 给所有应用设置了 PR_SET_NO_NEW_PRIVS，
  setuid 位提权被内核直接忽略；
- `Seccomp: 2`：seccomp 过滤模式拦截 setuid/setgid/setgroups 系统调用，返回 EPERM。

结论：应用进程内任何"借 setuid 提权"的路径（su、自研 runconuid 等）在该固件上都是死路。

### 路线 3（最终）：root 身份常驻 + 回环 socket

绕开"应用自己提权"，改为：**让 init 在开机时以 root 启动一个守护，应用只做网络客户端**。
应用进程本身永远不需要提权，只需 `INTERNET` 权限连接 127.0.0.1。

---

## 三、最终方案组成

### 3.1 init 服务定义

文件：`/system/etc/init/hh_rootd.rc`

```
service hh_rootd /system/xbin/hh_rootd.sh
    class main
    user root
    group root
```

### 3.2 守护脚本

文件：`/system/xbin/hh_rootd.sh`（标签 `u:object_r:system_file:s0`，权限 755）

```sh
#!/system/bin/sh
while true; do
  toybox nc -s 127.0.0.1 -p 9753 -l /system/bin/sh
  sleep 1
done
```

要点：

- 用设备自带 toybox 的 nc。其语法是 **flags 在前、-l 后紧跟命令**：
  `nc -s <addr> -p <port> -l <COMMAND>`；
  `-lL` 连用会报 `No 'L' with 'l'`。
- nc 是单连接监听器，连接结束后进程退出，靠外层 `while true` 重新拉起；
  快速连续调用会有短暂监听空窗，客户端需要重试（见 3.4）。
- **脚本里不要写 `setenforce 0`**（早期版本写过）。最终 SELinux 策略只对 su 域
  permissive，全局保持 Enforcing，详见第五节。

### 3.3 持久化位置（SAR 注意事项）

该设备是 **system-as-root**，/system 不是独立挂载点：

- remount 系统根：`mount -o rw,remount /`
- remount vendor 分区：`mount -o rw,remount /vendor`（SELinux 预编译策略在这里）

### 3.4 应用端 RootBridge.java

工程位置：`mobile/src/main/java/com/example/superstart/RootBridge.java`

- 连接 `127.0.0.1:9753`，发送 `命令 + "\nexit\n"`，读到 EOF 返回 stdout。
- **4 次重试，间隔 400ms**，容忍 nc 重启空窗；连接超时 2s，读超时 20s。
- 脚本落盘用 **base64 单行传输**（`Base64.NO_WRAP`），避免 PowerShell/adb 流式
  stdin 与 cat 的 EOF 死锁：
  `echo <b64> | base64 -d > /data/local/tmp/netease_hk_v4.sh; chmod 755 ...`
- AndroidManifest 必须加 `<uses-permission android:name="android.permission.INTERNET"/>`，
  否则连回环 socket 也会 EACCES。

公共 API：`isRunning / inject / start / stop / isKugouDisabled /
disableKugouComponents / enableKugouComponents`。

---

## 四、SELinux 问题（本方案最核心的坑）

### 4.1 现象

Enforcing 下 init 拒绝启动服务，dmesg 反复出现：

```
init: cannot execve('/system/xbin/hh_rootd.sh'): Permission denied
avc: denied { transition } scontext=u:r:init:s0
    tcontext=u:r:su:s0 tclass=process permissive=0
init: Service 'hh_rootd' exited with status 127
```

注意：tcontext 已经是 `u:r:su:s0`，说明 `type_transition` 已经匹配，
**真正缺的是配套的 allow 规则**。SELinux 域转换是一个"规则组合拳"，
只写 type_transition 不够。

### 4.2 工具

magiskpolicy：从 Magisk APK 的 `lib/arm64-v8a/libmagiskpolicy.so` 提取，push 到
`/data/local/tmp/magiskpolicy`，chmod 755。

- `--live '<规则>'`：注入运行时内核策略；
- `--save <文件>`：导出当前运行时策略；
- `--load <文件>`：把策略文件加载回内核（用于刷机前预验证，见 6.2）。

### 4.3 最终补齐的完整规则链

```
type_transition init system_file process su
allow init su process { transition rlimitinh siginh sigchld sigkill }
allow su system_file file { entrypoint read execute getattr open }
typepermissive su
```

规则解释：

| 规则 | 作用 |
|---|---|
| type_transition init system_file process su | init 执行 system_file 标签文件时，新进程进入 su 域 |
| allow init su process transition | 允许域转换本身（缺它就是 4.1 的 denied） |
| rlimitinh / siginh | 子进程继承资源限制与信号处理的必需权限 |
| sigchld / sigkill | init 给服务发 SIGCHLD/SIGKILL（服务管理、重启）所需 |
| allow su system_file entrypoint | su 域只允许以带 entrypoint 权限的文件作为入口 |
| read / execute / getattr / open | 读取并执行脚本文件 |
| typepermissive su | su 域设为宽容：守护内执行的各种命令不再逐条被拒 |

效果：**全局保持 Enforcing，仅 root 守护所在的 su 域 permissive**，
比 `setenforce 0` 干净得多。

### 4.4 注入操作记录（实测成功）

```sh
MP=/data/local/tmp/magiskpolicy
$MP --live "type_transition init system_file process su"
$MP --live "allow init su process transition"
$MP --live "allow init su process rlimitinh"
$MP --live "allow init su process siginh"
$MP --live "allow init su process sigchld"
$MP --live "allow init su process sigkill"
$MP --live "allow su system_file file entrypoint"
$MP --live "allow su system_file file read"
$MP --live "allow su system_file file execute"
$MP --live "allow su system_file file getattr"
$MP --live "allow su system_file file open"
# typepermissive su 已在前期注入；如重新部署需补：
# $MP --live "permissive su"
```

每条返回码均为 0。注入后 `setenforce 1` + `setprop ctl.start hh_rootd`，
服务在 Enforcing 下正常启动，dmesg 无 hh_rootd 相关 denied。

---

## 五、策略持久化（重启不失效）

开机时 init 加载的是预编译策略文件：

```
/vendor/etc/selinux/precompiled_sepolicy
```

### 5.1 写入流程（带防砖预验证）

```sh
# 1. 导出当前"已验证可用"的运行时策略
$MP --save /data/local/tmp/new_sepolicy          # RC=0，700242 字节

# 2. 预验证：把该文件 load 回内核。能 load 成功才允许刷机
$MP --load /data/local/tmp/new_sepolicy          # RC=0
# 加载后立刻再测一次守护，确认系统没被搞坏
echo "id; exit" | toybox nc -w 3 127.0.0.1 9753  # 正常返回 uid=0

# 3. 备份原文件后覆盖
mount -o rw,remount /vendor
cp /vendor/etc/selinux/precompiled_sepolicy /data/local/tmp/precompiled_sepolicy.bak2
cp /data/local/tmp/new_sepolicy /vendor/etc/selinux/precompiled_sepolicy
chmod 644 /vendor/etc/selinux/precompiled_sepolicy
mount -o ro,remount /vendor
```

关键安全动作是第 2 步：**`--load` 与开机加载走同一套内核接口**，
load 成功 ≈ 开机能加载，把变砖风险降到最低。

备份：
- `/data/local/tmp/precompiled_sepolicy.bak`（早期备份）
- `/data/local/tmp/precompiled_sepolicy.bak2`（最终覆盖前备份）

### 5.2 设备策略文件结构（备查）

```
/system/etc/selinux/plat_sepolicy.cil            # 平台 CIL 文本策略
/vendor/etc/selinux/vendor_sepolicy.cil          # 厂商 CIL 文本策略
/vendor/etc/selinux/plat_pub_versioned.cil
/vendor/etc/selinux/precompiled_sepolicy         # 实际编译产物（改这个）
```

---

## 六、设备端文件清单

| 路径 | 说明 |
|---|---|
| `/system/etc/init/hh_rootd.rc` | init 服务定义 |
| `/system/xbin/hh_rootd.sh` | root socket 守护脚本（755, system_file） |
| `/vendor/etc/selinux/precompiled_sepolicy` | 已注入规则的持久化策略 |
| `/data/local/tmp/magiskpolicy` | SELinux 策略编辑工具（755） |
| `/data/local/tmp/netease_hk_v4.sh` | 方向盘媒体键桥接脚本（应用每次注入时刷新，755） |
| `/data/local/tmp/netease_hk.log` / `.raw` | 桥接转发日志 / 原始按键日志 |
| `/data/local/tmp/precompiled_sepolicy.bak2` | 原始策略备份（回滚用） |
| `/system/xbin/su` | 打补丁的 su（shell 可用，应用不可用，保留备用） |

回滚方法（如需要）：

```sh
mount -o rw,remount /vendor
cp /data/local/tmp/precompiled_sepolicy.bak2 /vendor/etc/selinux/precompiled_sepolicy
mount -o ro,remount /vendor
reboot
```

---

## 七、桥接脚本行为（netease_hk_v4.sh 要点）

- `logcat -v brief 'Launcher-library-common:V' '*:S'` 监听车机 Launcher 的
  `OnHKChange` 回调；
- keyCode 映射：**2=上一曲(previous)、3=下一曲(next)、12=播放暂停(play-pause)**；
- keyState 去抖：**state=0（按下）无条件转发；state=1（抬起）仅在 10 分钟
  GRACE 期（600000ms）内转发**，避免抬起事件重复触发；
- 通过 `dumpsys media_session` 解析当前活跃 MediaSession，
  目标白名单：`com.netease.cloudmusic.iot`、`com.netease.cloudmusic`、
  `com.kugou.android.auto`；
- 无活跃第三方播放（state=3）时跳过，不抢系统/蓝牙/U盘的媒体控制；
- 最终经 `/system/bin/media dispatch <key>` 下发。

酷狗两个抢控组件由应用经 root 守护成对 `pm disable`（缺一不可，
只禁 Service 会触发透明 Activity 自激重连导致屏闪）：

```
com.kugou.android.auto/com.kugou.auto.proxy.AutoSdkAIDLRemoteService
com.kugou.android.auto/com.kugou.auto.proxy.AutoSdkEmptyActivity
```

判断禁用状态时只在 `dumpsys package` 的 **disabledComponents:** 段内匹配，
不能全文匹配（类名在组件清单里本来就会出现）。

---

## 八、验证记录

### 8.1 重启前（Enforcing 手动验证）

- 规则注入后 `setenforce 1`，`setprop ctl.start hh_rootd` 一次启动成功；
- `echo "id; exit" | toybox nc -w 3 127.0.0.1 9753`
  返回 `uid=0(root) gid=0(root) context=u:r:su:s0`；
- dmesg 无 hh_rootd 相关 avc denied。

### 8.2 方向盘按键实测

桥接日志 `/data/local/tmp/netease_hk.log`：

```
FWD code=3  -> next       (com.netease.cloudmusic.iot)
FWD code=2  -> previous   (com.netease.cloudmusic.iot)
FWD code=12 -> play-pause (com.netease.cloudmusic.iot)
```

`.raw` 中可见 Launcher 同时上报 state=0/1，脚本按去抖策略只转发应转发的事件。

### 8.3 重启后全链路（最终验收）

reboot → 54 秒后设备上线 → `adb root`：

| 检查项 | 结果 |
|---|---|
| getenforce | **Enforcing**（新策略开机被正常加载） |
| hh_rootd 进程 | init 早期自启（sh + toybox nc 均在） |
| socket 测试 | uid=0(root) context=u:r:su:s0 |
| HunterHack 进程 | 开机自启（BootReceiver → DockService） |
| 桥接守护 | 开机时刻自动注入并运行（netease_hk_v4.sh ×2 + logcat） |
| 酷狗组件 | disabledComponents 中两个组件均在 |
| mobile_data | **0**（移动数据守护开机自动关闭） |

---

## 九、踩坑清单（按踩坑顺序）

1. **boot.img 是 Mediatek 私有 AVB1 证书**，公开 testkey 无法重签 → 放弃 Magisk 刷 boot。
2. **应用进程 NoNewPrivs=1 + Seccomp=2**，setuid 路线对应用完全无效。
3. **chcon 会清掉 setuid 位**（kernel kill_priv）：必须先 chcon 后 chmod。
4. **runconuid 失败时内部会执行 setenforce(1)**，曾把全局 Permissive 意外翻回 Enforcing，排查了很久。
5. **avc denied tcontext 已是 su 域**，说明 type_transition 生效，缺的是 allow 规则——
   别只盯着 type_transition。
6. **域转换是规则组合拳**：transition + rlimitinh + siginh + sigchld + entrypoint + 读写执行，缺一条都起不来。
7. **SAR 设备 remount 根分区**是 `mount -o rw,remount /`，不是 /system；/vendor 单独 remount。
8. **toybox nc 语法特殊**：flags 在前，`-l` 后直接跟命令；不支持 -lL 连用。
9. **PowerShell 调 adb shell 会吞管道符和引号**：含 `|`、`$`、复杂引号的命令一律写成
   .sh 文件 push 后 `sh 执行`（Out-File ascii 并去掉 \r）。
10. **应用 socket 也要 INTERNET 权限**，即使只连 127.0.0.1，否则 EACCES。
11. **adb root 可用（userdebug）但重启失效**：只能用于调试，永久方案必须靠 init 服务。

---

## 十、安全提示

`127.0.0.1:9753` 当前是**无鉴权 root shell**：设备上任何声明了 INTERNET 权限的应用
都能连接并以 root 执行任意命令；网络上不可达（仅绑定回环地址），外部攻击面小，
但本机恶意/被攻破应用可以直接获得 root。

如需收紧，可选：

- 守护端加简单 token 校验（连接第一行必须发送预置密钥，不符即断开）；
- 改用 UNIX domain socket + 文件权限/属组（应用与守护约定组，SELinux 再限连接域）；
- 守护内做命令白名单（只允许 pm / media / settings 等固定前缀）。

---

## 十一、一键重部署所需物料（重装/换机时）

PC 侧保留：

- `inject_rules.sh`：注入运行时 SELinux 规则；
- `persist_policy.sh`：save → load 预验证 → 覆盖 vendor 预编译策略；
- `fix_hh_rootd.sh`：写入无 setenforce 的守护脚本并重启服务；
- `netease_hk_v4.sh`：桥接脚本（已内置在 HunterHack APK 的 assets 中，应用自动下发）。

设备侧前置条件：

- userdebug 固件（支持 `adb root`，用于首次部署）；
- / 与 /vendor 可 remount rw。
