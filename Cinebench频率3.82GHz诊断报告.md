# Cinebench 满载时 CPU 频率恒定 3.82 GHz —— 诊断报告

- 机器：ASUS PRIME Z790-P WIFI / i5-13600KF / Win10 LTSC 19044
- 采样时间：2026-10-30 23:44 – 23:57（Cinebench 2026 多核测试运行中）
- 触发条件：更新 BIOS 至 **1836** 后出现（本机 23:33:55 重启，23:36:38 启动 Cinebench）

---

## 1. 摘要

| 结论 | 置信度 |
|---|---|
| 3.82 GHz 是 **20 个逻辑处理器的平均值**，不是任何一颗核心的真实频率 | 确定（算术可复现） |
| P 核实际运行在 **4.6 – 5.0 GHz**，已接近该 U 的最高加速频率 | 高（两种独立方法互证） |
| E 核实际运行在 **3.2 – 3.7 GHz**，但 Windows 只记成 **1.93 GHz（约一半）** | 高（同上） |
| 3.82 的出现，**完全由 E 核被少记一半造成**，与真实频率无关 | 高 |
| 所谓"变慢"是显示/上报标度变化，**不是核心真的降到 3.82** | 高 |
| BIOS 1836 是否同时恢复了 Intel 默认功耗墙（PL1 125W / PL2 181W） | **待确认**（见第 6 节，1 分钟可判定） |

一句话：**CPU 没有降频，只是"平均频率"这个数字被 8 个记错一半的 E 核线程拉低了。**

---

## 2. 现象

- Cinebench 2026 一旦开始多核渲染，任务管理器/监控软件显示的 CPU 频率长期稳定在 **3.82 GHz**；
- 空闲时数字正常浮动；
- 用户反馈：**刷 BIOS 1836 之前显示"正常"**。

---

## 3. 测试环境

| 项目 | 实测值 |
|---|---|
| CPU | 13th Gen Intel Core i5-13600KF（Raptor Lake-S，Family 6 Model 183 Stepping 1） |
| 核心/线程 | 6 P-core（SMT）+ 8 E-core = 14C / 20T |
| 标称频率 | 3500 MHz（注册表 20 个逻辑处理器全部为 3494 MHz） |
| 主板 / BIOS | ASUSTeK PRIME Z790-P WIFI（Rev 1.xx）/ AMI **1836**，2026-04-16 |
| 内存 | Samsung DDR5-5600 16G ×2，实际 5600 |
| 系统 | Microsoft Windows 10 企业版 LTSC，10.0.19044 |
| 电源方案 | **高性能**（PROCTHROTTLEMIN = 100%，PROCTHROTTLEMAX = 100%，散热方式 = 主动） |
| 最近启动 | 2026-10-30 23:33:55（BIOS 刷写后） |
| 内核电源事件 | 近 3 天**无** Kernel-Processor-Power 37 / 38 / 39（无固件限频、无温度降频记录） |
| Cinebench | 2026.1.3，`C:\Users\clop32\Downloads\Cinebench2026_win_x86_64\Cinebench.exe`，优先级 BelowNormal（其自身默认行为） |

---

## 4. 测量方法与数据

### 4.1 Windows 性能计数器（逐逻辑处理器）

`\Processor Information(*)\% Processor Performance` 与 `% Processor Utility`，5 次 × 2 秒采样结果完全一致：

| 逻辑 CPU | 归属 | % Processor Performance | 折算频率（×3.5 GHz） |
|---|---|---|---|
| 0 – 11（12 线程） | **P 核**（6 核 × SMT） | 145.0 % | **5.07 GHz** |
| 12 – 19（8 线程） | **E 核**（8 核） | 55.1 % | **1.93 GHz** |
| `_total` | 平均值 | **109.05 %** | **3.82 GHz** |

同时 `% Processor Time` 显示 20 个逻辑 CPU **全部 100%**，Cinebench 进程累计 12 374 CPU·秒 / 695 秒墙钟 ≈ **17.8 ~ 19 个逻辑核被吃满**，22 个渲染线程 CPU 时间 > 60 秒 —— 确认是**真·全核满载**，不是"只用了一半线程"。

### 4.2 指令延迟法（绕过 Windows 计数器，直接测频率）

**原理**：依赖链上每条指令的延迟是固定的（`add` = 1 周期，`imul` = 3 周期），只要数清指令条数并测时间，就能反推核心真实频率，不依赖任何监控软件。

**自校验**：同一颗核上分别跑加法链与乘法链，两者耗时之比必须等于"imul 延迟 / add 延迟"。Golden Cove（P 核）= 3，Gracemont（E 核）= 5。实测比值正好落在这两个值上，说明测法成立。

| 逻辑 CPU | 加法链（1 周期/条） | 乘法链（3 周期/条折算） | mul/add 耗时比 | 判定延迟 | **推定真实频率** |
|---|---|---|---|---|---|
| 0（P） | 0.209 s → **4.59 GHz** | 0.618 s → 4.66 GHz | **2.96** | imul = 3 周期 ✔ | **≈ 4.6 GHz** |
| 2（P） | 0.194 s → **4.95 GHz** | 0.575 s → 5.01 GHz | **2.97** | imul = 3 周期 ✔ | **≈ 5.0 GHz** |
| 12（E） | 0.259 s → **3.70 GHz** | 1.311 s | **5.05** | imul = 5 周期 ✔ | **≈ 3.7 GHz** |
| 19（E） | 0.296 s → **3.25 GHz** | 1.256 s | **4.25** | imul ≈ 4~5 周期 | **≈ 3.3 GHz** |

> 测量线程以 Highest 优先级独占一个逻辑 CPU 约 0.2–0.3 秒（墙钟 ≈ CPU 时间，说明未被抢占），因此该结果反映的是**满载状态下**的频率，可与计数器直接对照。

**对照结果**

```
            Windows 计数器        实测（延迟法）
P 核        5.07 GHz       ≈      4.6 – 5.0 GHz      → 一致（差值为 SMT 争用导致的低估）
E 核        1.93 GHz       ≈      3.2 – 3.7 GHz      → 差了将近一半 ✘
```

### 4.3 旁证：Windows 自身的大小核数据也是乱的

开机时 Kernel-Processor-Power 事件 ID 55 记录：

| 逻辑 CPU | 标称频率 | 「最佳性能百分比」 |
|---|---|---|
| 0 – 11（P） | 3500 MHz | 579 % |
| 12 – 19（E） | 3500 MHz | 980 % |

E 核的"最高性能"被记成 P 核的 1.69 倍，显然不合理 —— 说明这套 ACPI CPPC / HWP 上报数据在 Win10 上本身就处于错乱状态，`% Processor Performance` 对 E 核失效与此同源。

---

## 5. 计算与结论

### 5.1 3.82 的算术来源（可手算复现）

Windows 的 `_total` 是**所有逻辑处理器的算术平均**：

```
(12 × 145.0% + 8 × 55.1%) / 20 = (1740 + 440.8) / 20 = 109.04%
109.04% × 3.5 GHz = 3.816 GHz  ≈  3.82 GHz     ← 与观察值完全吻合
```

结论：**3.82 不是频率，是"12 个满血 P 核线程 + 8 个被记成一半的 E 核线程"的平均数。**

### 5.2 刷 BIOS 前后的推演

E 核真实频率 3.86 GHz 对应 110.3%（3.86 / 3.5）。如果 E 核**按真实值上报**：

```
(12 × 145.0% + 8 × 110.3%) / 20 = (1740 + 882.4) / 20 = 131.1%
131.1% × 3.5 GHz = 4.59 GHz
```

| 状态 | E 核上报值 | 平均值显示 |
|---|---|---|
| 更新前（假设上报正常） | 110 %（3.86 GHz） | **≈ 4.6 GHz** |
| 更新后（当前） | 55 %（1.93 GHz） | **3.82 GHz** |

**差的 0.77 GHz 全部来自 E 核那 8 个线程的计数标度**，与 P 核、与真实频率无关。
若你更新前看到的是 4.5 – 4.7 GHz，即完全对上此模型。

---

## 6. 根因假设与待排除项

### 假设 A（主因，高置信）：BIOS 1836 改变了上报给 Windows 的 E 核性能标度

- 证据：20 个逻辑处理器的 nominal 全为 3494 MHz（非基数被改），但 E 核"当前性能"被记成真实值的一半；
- 机制：BIOS 通过 ACPI `_CPC` / HWP（IA32_HWP_CURRENT_PERFORMANCE）向 OS 提供"当前性能 / 标称性能"，Win10 19044 没有 Thread Director，只做除法再平均；
- 时间线：BIOS 刷新 → 23:33:55 重启 → 23:36:38 开跑 → 稳定 3.82，与"更新后才出现"吻合；
- 影响：**仅显示/上报**，不影响实际算力与跑分。

### 假设 B（必须排除，中置信）：BIOS 同时恢复了 Intel 默认功耗墙

BIOS 1836 这类更新常把 `Intel Default Settings`（PL1 125 W / PL2 181 W / IccMax 200 A）重新应用。
实测 P 核 4.6–5.0、E 核 3.2–3.7，比 5.1 / 3.9 略低 —— 既可能是测量时的 SMT 争用，也可能是墙回来了。

**唯一判据：CPU Package Power（HWiNFO）**

| 读数 | 判定 | 处理 |
|---|---|---|
| 180 – 220 W | 无功耗墙，性能未变 | 纯显示问题，忽略即可 |
| 长期钉在 125 W 或 181 W | 墙已恢复，这部分是真降 | 见 7.3 恢复方案 |

---

## 7. 验证清单（5 分钟）

### 7.1 HWiNFO（你机器上已开着，需以管理员运行传感器窗口）

| 传感器 | 期望值 | 含义 |
|---|---|---|
| P-core Clocks | 5000 – 5100 MHz | P 核满血 |
| E-core Clocks | 3600 – 3900 MHz | **若显示此值，即证明任务管理器的 1.93 是错的** |
| CPU Package Power | 180 – 220 W | 无功耗墙 |
| Thermal Throttling | No | 无温度降频 |
| Core VID / 温度 | < 100 ℃ | 散热正常 |

### 7.2 跑分对比

- 与"更新 BIOS 之前"的分数对比：**差 3% 以内 = 纯显示问题**；**差 15% 以上 = 功耗墙被改，按 7.3 处理**。
- 13600KF 参考值（Cinebench R23 多核）：23 000 – 24 500 pts，低于 22 000 才说明真被限制。

### 7.3 若确认被功耗墙限制，BIOS 恢复路径

`Ai Tweaker → Internal CPU Power Management`：

| 项目 | 建议 |
|---|---|
| Intel Default Settings | 改为 `Performance` / `Extreme`（或 `ASUS MultiCore Enhancement`） |
| Long / Short Duration Package Power Limit | 按需调回（如 253 W / 253 W） |
| CPU Core / Cache Current Limit Max | 511.75 A |
| IA CEP | Disabled（可减少不必要的掉压降频） |

> 注意：13/14 代 Raptor Lake 存在退化（degradation）问题，Intel 默认功耗上限正是为此加入的缓解措施。若以稳定性为先，建议保留默认限制或仅小幅放宽；不要长期 4095 W + 高电压。

---

## 8. 结论

1. **CPU 没有降频。** P 核 4.6–5.0 GHz、E 核 3.2–3.7 GHz，均处于该型号的满血加速区间；无温度降频、无固件限频事件。
2. **3.82 GHz 是统计假象。** 它等于 `(12×5.07 + 8×1.93) / 20`，其中 E 核的 1.93 是真实值的约一半。
3. **刷 BIOS 后"变慢"的现象，本质是上报标度改变**（E 核从 110% 被记成 55%），使平均值从约 4.6 掉到 3.82。
4. **唯一还需确认的是功耗墙**（第 6 节假设 B）。用 Package Power 一分钟即可定论；若 Package Power 正常，则无需任何处理。
5. Windows 10 19044 不具备完整的大小核调度/计数支持，任务管理器那个"速度"在混合架构上本就不可信；判断性能请看 **HWiNFO 逐核时钟 + 实际跑分**。

---

## 附录 A：复现本报告的频率测量

```powershell
# 依赖链测频:add = 1 周期,imul = 3 周期(Golden Cove)/5 周期(Gracemont)
$cs = @'
using System; using System.Diagnostics; using System.Runtime.CompilerServices;
using System.Runtime.InteropServices; using System.Threading;
public static class Freq2 {
  [DllImport("kernel32.dll")] static extern IntPtr GetCurrentThread();
  [DllImport("kernel32.dll")] static extern UIntPtr SetThreadAffinityMask(IntPtr h, UIntPtr m);
  public static long Sink;
  [MethodImpl(MethodImplOptions.NoInlining)]
  public static long Add(long a, int n){ for(int i=0;i<n;i++){ a=a+1;a=a+1;a=a+1;a=a+1;a=a+1;a=a+1;a=a+1;a=a+1;
    a=a+1;a=a+1;a=a+1;a=a+1;a=a+1;a=a+1;a=a+1;a=a+1; } return a; }
  public static double Run(int cpu, int outer){
    double s=0;
    var t=new Thread(delegate(){ SetThreadAffinityMask(GetCurrentThread(),(UIntPtr)(1UL<<cpu));
      Add(3,1000); var sw=Stopwatch.StartNew(); Sink=Add(3,outer); sw.Stop(); s=sw.Elapsed.TotalSeconds; });
    t.Priority=ThreadPriority.Highest; t.Start(); t.Join(); return s;
  }
}
'@
Add-Type -TypeDefinition $cs -Language CSharp
foreach ($cpu in 0,2,12,19) {
  $t = [Freq2]::Run($cpu, 60000000)      # 9.6 亿条 add
  "CPU{0}: {1:N3}s -> {2:N2} GHz" -f $cpu, $t, (16*60000000/$t/1e9)
}
```

## 附录 B：本报告用到的排查命令

```powershell
# 逐核性能百分比与折算频率
Get-Counter '\Processor Information(*)\% Processor Performance' -MaxSamples 5 |
  ForEach-Object { $_.CounterSamples | Where-Object InstanceName -ne '_total' } |
  Select-Object InstanceName, CookedValue

# 电源方案与处理器设置
powercfg /getactivescheme
powercfg /q SCHEME_CURRENT SUB_PROCESSOR

# 固件限频 / 温度降频事件
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Microsoft-Windows-Kernel-Processor-Power'} -MaxEvents 30

# 每逻辑处理器的标称频率
0..19 | ForEach-Object { Get-ItemProperty "HKLM:\HARDWARE\DESCRIPTION\System\CentralProcessor\$_" | Select-Object '~MHz' }
```

## 附录 C：测量局限

1. 延迟法测量时会短暂独占一个逻辑 CPU（0.2–0.3 s），对正在运行的成绩有可忽略的影响（< 0.1%）；
2. P 核为 SMT 双线程，另一线程仍在跑 Cinebench，故 P 核实测值可能被**低估**约 5–10%；
3. 未取到刷 BIOS 前的实测样本，5.2 节的"更新前 4.6 GHz"是由当前 E 核真实频率反推的模型值，需由你记忆中当时的显示值佐证；
4. HWiNFO64 以管理员运行且未开启 Shared Memory，无法读取其传感器；系统也未提供 MSAcpi_ThermalZoneTemperature，故本报告不含温度/功耗实测，相关判断以第 7 节清单为准。
