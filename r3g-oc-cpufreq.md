# R3G (MT7621) OC 与动态调频说明

## 目标
- 分支：`openwrt-25.12-r3g-plus-oc`
- 设备：`ramips/mt7621`（Xiaomi Mi Router 3G）
- 能力：
  - 支持 MT7621 CPU 超频档位
  - 支持基于 governor 的按需调频（`schedutil`）
  - 保留 `performance` 用于固定高频

## 频率档位与相对性能

默认频率基准：`880 MHz = 100%`

| 频率 | 相对性能（按频率线性估算） | 说明 |
|---|---:|---|
| 880 MHz | 100.00% | 默频 |
| 1000 MHz | 113.64% | 轻度 OC |
| 1100 MHz | 125.00% | 中度 OC |

> 公式：`性能% = 频率 / 880 * 100`

## 内核实现点

本分支通过内核补丁新增 MT7621 cpufreq 驱动：

- 补丁：`target/linux/ramips/patches-6.12/313-cpufreq-add-mt7621-driver.patch`
- 依赖修复：`target/linux/ramips/patches-6.12/314-mips-ralink-mt7621-enable-cpufreq-prereqs.patch`
- 新增驱动：`drivers/cpufreq/mt7621-cpufreq.c`
- Kconfig 开关：`CONFIG_MT7621_CPUFREQ`

驱动核心逻辑：

1. 通过 `syscon` 读取 MT7621 时钟参数（xtal、分频、PLL 配置）。
2. 将目标频率（880/1000/1100）转换为 CPU PLL `FBDIV`。
3. 动态写入 `MEMC_REG_CPU_PLL` 完成频率切换。
4. 通过 cpufreq 框架暴露标准 governor 接口给用户态。

### 为什么之前 LuCI 显示“不支持”

根因不是 LuCI 包，而是内核 Kconfig 依赖链：

- MIPS 里 `drivers/cpufreq/Kconfig` 只有在  
  `CPU_SUPPORTS_CPUFREQ && MIPS_EXTERNAL_TIMER` 条件满足时才会被引入。
- MT7621 默认未选中这两个前置符号，导致  
  `CONFIG_CPU_FREQ/CONFIG_MT7621_CPUFREQ` 在最终内核 `.config` 里被丢弃。
- 结果就是系统没有 `/sys/devices/system/cpu/cpufreq`，LuCI 才会报设备不支持。

`314` 补丁在 `SOC_MT7621` 下补齐了：

- `select CPU_SUPPORTS_CPUFREQ`
- `select MIPS_EXTERNAL_TIMER`

这样 cpufreq 子系统会真正进内核，LuCI 页面和 `/etc/init.d/cpufreq get_policies` 才能正常工作。

## governor 策略

当前仅保留两个 governor：

- `schedutil`：按负载动态切频（推荐默认）
- `performance`：固定高频

为什么不单独增加“默频 governor”：

- “默频”本质是频率上下限策略，不是 governor 类型。
- 若要固定默频，可直接设置：
  - `governor=schedutil`
  - `minfreq=maxfreq=880000`

## 默认用户态配置

`package/emortal/cpufreq/files/cpufreq.uci` 已增加 `ramips/mt7621` 默认策略：

- governor：`schedutil`
- 频率范围：`880000 ~ 1100000`

说明：
- 默认先给 1.1GHz 动态档，兼顾性能与稳定性。
- 本版为稳定性考虑，最高频率收敛到 1.1GHz（不再放开 1.2GHz）。

## 运行时切换示例

切到固定高频（`performance`）：

```sh
uci set cpufreq.@cpufreq[0].governor='performance'
uci commit cpufreq
/etc/init.d/cpufreq restart
```

切回按需调频（`schedutil`）并放开到 1.1GHz：

```sh
uci set cpufreq.@cpufreq[0].governor='schedutil'
uci set cpufreq.@cpufreq[0].minfreq='880000'
uci set cpufreq.@cpufreq[0].maxfreq='1100000'
uci commit cpufreq
/etc/init.d/cpufreq restart
```

固定默频（不新增第三个 governor）：

```sh
uci set cpufreq.@cpufreq[0].governor='schedutil'
uci set cpufreq.@cpufreq[0].minfreq='880000'
uci set cpufreq.@cpufreq[0].maxfreq='880000'
uci commit cpufreq
/etc/init.d/cpufreq restart
```

## 相关构建配置

`target/linux/ramips/mt7621/config-6.12` 已启用：

- `CONFIG_CPU_FREQ=y`
- `CONFIG_MT7621_CPUFREQ=y`
- `CONFIG_CPU_FREQ_GOV_PERFORMANCE=y`
- `CONFIG_CPU_FREQ_GOV_SCHEDUTIL=y`
- `CONFIG_CPU_FREQ_DEFAULT_GOV_SCHEDUTIL=y`

并显式关闭其他 governor（ondemand/conservative/powersave/userspace）。
