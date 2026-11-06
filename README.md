# SystemHelper 免 Root 停用方案：小米 17 Pro Max 实测记录（公开版）

本文保留复现所需的机型、系统版本与应用版本，已去除真实设备序列号、个人电脑路径和精确操作时间。

## 1. 结论与适用范围

**本机已成功在不获取 Root 权限的情况下，将 SystemHelper 从主用户卸载，并通过了一次完整重启验证。**

这里的“停用”准确地说是**取消当前用户的安装状态**，结果为 `installed=false`，不是仅限制后台，也不是删除系统分区里的 APK。系统原始应用文件仍然保留，可通过 `install-existing` 恢复。

| 项目 | 本次实测值 |
| --- | --- |
| 手机 | 小米 17 Pro Max，型号 2509FPN0BC |
| 系统版本 | 澎湃 OS4.0.6.0.XPBCNXM |
| Android 版本 | 17 |
| 目标应用 | SystemHelper |
| 目标包名 | `com.mobiletools.systemhelper` |
| 应用版本 | `4.0.0-20260910`，versionCode 34 |
| 当前用户 | 0 |
| 执行身份 | ADB shell，UID 2000 |

本次没有获取 Root，也没有执行解锁引导加载程序。此方法只在上述手机和系统版本上验证过；未来系统升级、恢复出厂设置后的状态尚未验证。

注意：不要与手机里的 `com.qualcomm.qti.services.systemhelper` 混淆，本次仅处理 `com.mobiletools.systemhelper`。

## 2. 为什么单纯限制后台不够

最初执行过以下命令：

```text
adb shell cmd appops set --user 0 com.mobiletools.systemhelper RUN_ANY_IN_BACKGROUND ignore
adb shell am force-stop --user 0 com.mobiletools.systemhelper
```

第一次重启后的实测结果：

- `RUN_ANY_IN_BACKGROUND` 仍为 `ignore`，后台限制确实保留。
- 重启后，系统电话组件通过 `SIMStateReceiver` 启动了该应用。
- 约 24 秒后，系统因内存压力结束了该进程。

因此，**后台限制和强制停止不能阻止本机上的所有启动入口**。当下看不到进程，也不能据此认定它从未在开机时运行。

随后尝试的常规方式也被拒绝：

| 方式 | 实测反馈 |
| --- | --- |
| 整包 `pm disable-user --user 0` | `Cannot disable system packages.` |
| `pm uninstall -k --user 0` | `only root can delete system app for a particular user` |
| `pm hide --user 0` | 缺少 `android.permission.MANAGE_USERS` |
| 单独禁用 `SIMStateReceiver` | `Shell cannot change component state ...` |

这些失败尝试均未改变目标应用的安装或启用状态。

## 3. 成功方案

使用一个临时 Java / DEX 工具，在普通 ADB shell 身份下直接调用本机仍然可用的应用管理接口：

```java
IPackageManager.deletePackageAsUser(
    "com.mobiletools.systemhelper",
    -1,
    null,
    0,
    DELETE_KEEP_DATA | DELETE_SYSTEM_APP
);
```

参数含义：

- 固定目标包为 `com.mobiletools.systemhelper`。
- `-1`：不限定版本号。
- `null`：不接收旧式卸载回调；实际结果必须通过系统状态回读验证。
- `0`：仅处理主用户。
- `DELETE_KEEP_DATA = 1`：请求保留应用数据。
- `DELETE_SYSTEM_APP = 4`：执行系统应用的用户级卸载。
- 合并标志值为 `5`。

**实际观察是：普通卸载命令被拒绝，直接接口调用成功。不能据此推广为所有 Android 17 手机都可使用。** 接口调用返回只代表请求已提交，不能单凭返回信息认定成功。

## 4. 复现步骤（Windows PowerShell）

以下命令仅处理 `com.mobiletools.systemhelper` 的主用户安装状态。先确认手机系统版本和包名；已经显示 `installed=false` 时无需重复卸载。此处是实测过程的通用写法，并未在每一种手机上验证。

### 4.1 准备工具与源码

需要 Android Platform-Tools（含 ADB）、JDK（含 `java`、`javac`）和 R8 8.3.37（含 D8）。平台工具与 R8 应从其官方来源获取；本文不附带 APK、私人备份或原始日志。

在自己的电脑上准备以下目录；路径由环境变量生成，不包含作者的个人路径：

```powershell
$workDirectory = Join-Path $env:TEMP 'systemhelper-tools'
New-Item -ItemType Directory -Path $workDirectory -Force | Out-Null
$adbExecutable = Join-Path $workDirectory 'platform-tools\adb.exe'
$r8Path = Join-Path $workDirectory 'r8-8.3.37.jar'
```

将 Platform-Tools 解压到 `$workDirectory`，确保存在 `platform-tools\adb.exe`；将 R8 保存为 `r8-8.3.37.jar`。把本文附录源码保存为该目录下的 `SystemHelperControl.java`（UTF-8 编码），并确保 `java`、`javac` 可从 PowerShell 运行。

开启手机 USB 调试，连接并授权。下面使用 `-d` 选择唯一一台通过 USB 连接的设备，命令中不需要写入真实序列号。只连接准备处理的这一台手机。

```powershell
& $adbExecutable -d get-state
if ($LASTEXITCODE -ne 0) { throw '请确认只连接一台 USB 调试设备，且已允许调试授权。' }

& $adbExecutable -d shell id
& $adbExecutable -d shell am get-current-user
& $adbExecutable -d shell getprop ro.build.version.incremental
& $adbExecutable -d shell dumpsys package com.mobiletools.systemhelper |
    Select-String -Pattern 'versionCode=|versionName=|^    User 0: ceData' -Context 0,1
```

应为 `uid=2000(shell)`，当前用户为 `0`；若不符合，停止，不直接套用后续操作。系统状态输出只用于本地核对，不要未经检查直接粘贴到公开仓库。

### 4.2 编译临时工具

在同一个 PowerShell 会话中执行：

```powershell
& javac --release 8 -encoding UTF-8 -d $workDirectory (Join-Path $workDirectory 'SystemHelperControl.java')
if ($LASTEXITCODE -ne 0) { throw 'Java 编译失败。' }

& java -cp $r8Path com.android.tools.r8.D8 --min-api 26 --output $workDirectory (Join-Path $workDirectory 'SystemHelperControl.class')
if ($LASTEXITCODE -ne 0) { throw 'DEX 转换失败。' }
```

输出为 `$workDirectory` 下的 `classes.dex`。

### 4.3 检查接口并执行用户级卸载

```powershell
& $adbExecutable -d push (Join-Path $workDirectory 'classes.dex') /data/local/tmp/systemhelper-control.dex
if ($LASTEXITCODE -ne 0) { throw '临时工具传输失败。' }

& $adbExecutable -d shell chmod 444 /data/local/tmp/systemhelper-control.dex
if ($LASTEXITCODE -ne 0) { throw '设置临时工具权限失败。' }

& $adbExecutable -d shell 'CLASSPATH=/data/local/tmp/systemhelper-control.dex app_process /system/bin SystemHelperControl inspect'
if ($LASTEXITCODE -ne 0) { throw '接口检查失败，不继续执行。' }

& $adbExecutable -d shell 'CLASSPATH=/data/local/tmp/systemhelper-control.dex app_process /system/bin SystemHelperControl remove-for-user'
if ($LASTEXITCODE -ne 0) { throw '卸载请求失败，请查看返回信息。' }
```

`inspect` 只检查身份和接口，不修改安装状态。执行 `remove-for-user` 后，实测返回：

```text
UID=2000
PACKAGE=com.mobiletools.systemhelper USER=0 KEEP_DATA=1 DELETE_SYSTEM_APP=4
REQUEST_SENT_VERIFY_PACKAGE_STATE
```

这些是身份和请求参数，不是卸载成功回调；接下来必须回读状态。

### 4.4 验证并重启复查

```powershell
# 当前用户的已安装列表应不再出现该包。
& $adbExecutable -d shell pm list packages --user 0 com.mobiletools.systemhelper

# 关注主用户这一行，应包含 installed=false。
& $adbExecutable -d shell dumpsys package com.mobiletools.systemhelper |
    Select-String -Pattern '^    User 0: ceData' -Context 0,1

# SIM 状态和开机广播应找不到接收器。
& $adbExecutable -d shell cmd package query-receivers --components --user 0 -a android.intent.action.SIM_STATE_CHANGED -p com.mobiletools.systemhelper
& $adbExecutable -d shell cmd package query-receivers --components --user 0 -a android.intent.action.BOOT_COMPLETED -p com.mobiletools.systemhelper

# 注册服务应找不到，活动服务应为空。
& $adbExecutable -d shell cmd package query-services --components --user 0 -a com.mobiletools.systemhelper.service.IRegisterHelperService -p com.mobiletools.systemhelper
& $adbExecutable -d shell dumpsys activity services com.mobiletools.systemhelper

# 检查该包的进程以及带冒号后缀的子进程，应无匹配项。
& $adbExecutable -d shell ps -A |
    Select-String -Pattern 'com\.mobiletools\.systemhelper(?::[^\s]+)?\s*$'
```

手动重启手机、解锁并重新连接后，先确认 `sys.boot_completed` 为 `1`，再重复上述检查。

```powershell
& $adbExecutable -d shell getprop sys.boot_completed

# 查看当前开机日志中的目标启动事件；日志可能不完整，应结合安装状态判断。
& $adbExecutable -d logcat -b events -d -v threadtime |
    Select-String -Pattern 'am_proc_start.*com\.mobiletools\.systemhelper[,:\]]'
```

没有找到进程或日志不单独构成充分证明，重点是用户安装状态仍为 `installed=false`，广播接收器和服务也无法解析。

核对完成后清理临时 DEX：

```powershell
& $adbExecutable -d shell rm -f /data/local/tmp/systemhelper-control.dex
if ($LASTEXITCODE -ne 0) { throw '临时工具清理失败。' }
```

## 5. 本次重启验证结果

成功执行接口调用后再次重启，待系统完成开机并解锁后检查，实测结果如下：

| 检查项 | 实测结果 |
| --- | --- |
| 主用户 0 | `installed=false`、`stopped=true` |
| 主用户已安装列表 | 不包含目标包 |
| SIM 状态广播 | `No receivers found` |
| 开机广播 | `No receivers found` |
| 注册服务 | `No services found` |
| 活动服务 | `(nothing)` |
| 当前进程 | 未发现目标进程 |
| 本次开机的进程事件日志 | 未发现目标启动记录 |
| 系统原始 APK | 仍在 `/system/priv-app/com.mobiletools.systemhelper/` |

这组结果验证了**本次完整重启后仍维持用户级卸载状态**。它不等于对未来系统更新、恢复出厂设置或所有系统行为作永久保证。

## 6. 如何恢复

通过以下命令重新登记系统中已有的 APK。本文记录的实测过程未实际执行恢复，以保持卸载状态；恢复后应自行回读安装状态验证。

```powershell
$workDirectory = Join-Path $env:TEMP 'systemhelper-tools'
$adbExecutable = Join-Path $workDirectory 'platform-tools\adb.exe'

& $adbExecutable -d shell pm install-existing --user 0 com.mobiletools.systemhelper
if ($LASTEXITCODE -ne 0) { throw '恢复安装失败。' }

# 保留本文先前设置的后台限制。
& $adbExecutable -d shell cmd appops set --user 0 com.mobiletools.systemhelper RUN_ANY_IN_BACKGROUND ignore
if ($LASTEXITCODE -ne 0) { throw '恢复后台限制失败。' }

& $adbExecutable -d shell pm list packages --user 0 com.mobiletools.systemhelper
```

若还需要撤销后台限制，应恢复自己操作前记录的值。本文实测设备的初始值为 `allow`，仅在自己的原值也为 `allow` 时执行：

```powershell
& $adbExecutable -d shell cmd appops set --user 0 com.mobiletools.systemhelper RUN_ANY_IN_BACKGROUND allow
```

原值为默认状态时，应改用 `default`。

## 7. 关于隐私与验证边界

本次曾读取应用权限、权限调用记录、进程和服务状态。尝试读取应用私有目录时返回 `Permission denied`，`run-as` 返回应用不可调试。因此，没有读取其私有数据库，也没有获取实际上传报文，不能据此断言已经泄露了哪些具体个人数据。

本方案的结果是阻止当前用户继续正常运行该包，不能撤回可能已经上传的数据。执行时传入了保留数据标志，但由于私有目录不可读，没有逐项检查内部数据是否保留。

本文件记录本机实测，不对第三方审计仓库的全部结论作独立背书。系统升级或恢复出厂设置后，应重新检查用户安装状态。

## 附录：临时工具完整源码

文件名必须为 `SystemHelperControl.java`。源码固定目标包和用户编号，默认只检查接口，仅在参数为 `remove-for-user` 时提交操作。

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

/** 仅检查或移除主用户的 SystemHelper 安装登记，保留数据和系统原始文件。 */
public final class SystemHelperControl {
    public static void main(String[] arguments) throws Exception {
        int uid = (Integer) Class.forName("android.os.Process").getMethod("myUid").invoke(null);
        if (uid != 2000) throw new IllegalStateException("Expected ADB shell UID 2000, got " + uid);
        Class<?> packageManagerInterface = Class.forName("android.content.pm.IPackageManager");
        Object packageManager = Class.forName("android.app.ActivityThread")
                .getMethod("getPackageManager").invoke(null);
        Class<?> packageManagerConstants = Class.forName("android.content.pm.PackageManager");
        int keepData = packageManagerConstants.getField("DELETE_KEEP_DATA").getInt(null);
        int systemApp = packageManagerConstants.getField("DELETE_SYSTEM_APP").getInt(null);
        Method deleteMethod = packageManagerInterface.getMethod("deletePackageAsUser",
                String.class, int.class, Class.forName("android.content.pm.IPackageDeleteObserver"),
                int.class, int.class);
        System.out.println("UID=" + uid);
        System.out.println("METHOD=" + deleteMethod);
        System.out.println("PACKAGE=com.mobiletools.systemhelper USER=0 KEEP_DATA=" + keepData
                + " DELETE_SYSTEM_APP=" + systemApp);
        if (arguments.length == 0 || !"remove-for-user".equals(arguments[0])) return;
        try {
            deleteMethod.invoke(packageManager, "com.mobiletools.systemhelper", -1, null, 0,
                    keepData | systemApp);
            // 接口异步执行；需在主机回读安装状态，不能把调用返回视为卸载成功。
            System.out.println("REQUEST_SENT_VERIFY_PACKAGE_STATE");
        } catch (InvocationTargetException exception) {
            exception.getCause().printStackTrace();
            System.exit(1);
        }
    }
}
```

保存上述源码后，按第 4 节准备工具、编译和执行。

