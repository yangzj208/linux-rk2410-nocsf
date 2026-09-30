# linux-rk2410-nocsf（ca-a2b）

本 fork 在 Radxa `6.1.84-18`（`radxa/kernel` @ `fd36690`）上增加 ca-a2b 所需的 MCP251xFD 与 PCM6240。包版本 **`6.1.84-18.ca.a2b1`**。装上后 `uname -r` 应为：

```text
6.1.84-18.ca.a2b1-rk2410-nocsf
```

不要对上游 radxa 提 PR，也不要把 deb 发到第三方。交付物只在本仓。

## 构建

```bash
git clone --recurse-submodules https://github.com/yangzj208/linux-rk2410-nocsf.git
cd linux-rk2410-nocsf
git checkout ca-a2b-pcm-mcp
make deb
```

`KERNEL_DEFCONFIG` 末尾是 `ca_a2b.config`，fragment 会在官方 `radxa.config` / `radxa_custom.config` 之后合并。产物在仓库上一级目录，至少包括：

- `linux-image-6.1.84-18.ca.a2b1-rk2410-nocsf_6.1.84-18.ca.a2b1_arm64.deb`
- `linux-image-rock-4d_6.1.84-18.ca.a2b1_all.deb`
- `linux-image-radxa-rk3576_6.1.84-18.ca.a2b1_all.deb`

ROCK 4D 与 generic RK3576 共用同一套 image 包，板级差异在 meta 包和设备树 overlay。

## 安装

在板子上（已有 Radxa 源、能装到 `radxa-overlays-dkms`）：

```bash
sudo apt install \
  ./linux-image-6.1.84-18.ca.a2b1-rk2410-nocsf_6.1.84-18.ca.a2b1_arm64.deb \
  ./linux-headers-6.1.84-18.ca.a2b1-rk2410-nocsf_6.1.84-18.ca.a2b1_arm64.deb \
  ./linux-image-rock-4d_6.1.84-18.ca.a2b1_all.deb \
  ./linux-image-radxa-rk3576_6.1.84-18.ca.a2b1_all.deb
sudo reboot
```

只上 ROCK 4D 时可以不装 `linux-image-radxa-rk3576` meta 包，反过来也一样。image / headers 两个 arm64 包都要装。

## Overlay

内核只提供 `mcp251xfd` 与 `snd-soc-pcm6240`。板 B 的设备树 overlay 用文档仓：

https://github.com/yangzj208/custom_ca_a2b/pull/5

按该 PR 安装 overlay。复位脚在 DT 里写成 `reset-gpios`（驱动 con_id 是 `reset`）。固件 bin 缺失或复位 GPIO 探测失败时驱动只告警，声卡仍然注册。不要改回 dummy codec，也不要用手写 `i2cset` 代替驱动。

## 上板前自检

deb 解开后先确认模块和配置在包里，再上板看运行时：

```bash
# 包内
dpkg-deb -I linux-image-6.1.84-18.ca.a2b1-rk2410-nocsf_*_arm64.deb
dpkg-deb -c linux-image-6.1.84-18.ca.a2b1-rk2410-nocsf_*_arm64.deb \
  | grep -E 'mcp251xfd|snd-soc-pcm6240|config-'

# 上板、重启后
uname -r
grep -E 'CONFIG_CAN_MCP251X=|CONFIG_CAN_MCP251XFD=|CONFIG_SND_SOC_PCM6240=|CONFIG_SND_SOC_ROCKCHIP_SAI=|CONFIG_SND_SIMPLE_CARD=' \
  /boot/config-$(uname -r)
modinfo mcp251xfd
modinfo snd-soc-pcm6240
```

预期：`uname -r` 含 `6.1.84-18.ca.a2b1-rk2410-nocsf`；`CONFIG_CAN_MCP251X=m`、`CONFIG_CAN_MCP251XFD=m`、`CONFIG_SND_SOC_PCM6240=m`；两个 `modinfo` 都能找到模块。

## 验收

overlay 生效并加载模块后：

- `arecord -l` 能看到 PCM6240 这张声卡
- CAN 口出现（`ip link show type can`，或 `dmesg` 里有 `mcp251xfd`）
