# 用 ESP32 让 Wi‑Fi 版 iPad 切换到香港监管域并启用 AirPods 助听功能

这是一份简化版实测教程。测试环境为：

- Wi‑Fi 版 iPad
- iPadOS 27 正式版（24A437）
- AirPods Pro 3
- 普通 ESP32 开发板
- 一台 Mac

最终效果是 iPad 的监管域从 `CN` 更新为 `HK`，随后出现“听力健康”和“助听器”设置。完成配置后，同一副 AirPods 在同一 Apple 账户的 Mac 和 iPhone 上也可以同步相关设置。

## 一、原理

iPad 不会只根据公网 IP 判断地区。系统的 `countryd` 会综合：

- 公网 IP 对应的地区（GeoIP）
- Wi‑Fi Beacon 中的国家代码（Country IE）
- 周围 Wi‑Fi 的 BSSID 定位结果
- 设备已有的位置记录

因此，仅使用香港 VPN 通常不够。本方案让 ESP32 广播一组被苹果定位服务识别为香港的 BSSID，同时在 Beacon 中写入香港国家代码 `HK`，让多个判断来源保持一致。

## 二、需要准备

1. 一台 Wi‑Fi 版 iPad
2. AirPods Pro 2 或 AirPods Pro 3
3. ESP32 开发板
4. Mac 和 USB 数据线
5. 一组来自香港同一小范围、仍然有效的 BSSID
6. 周围 Wi‑Fi 和 GPS 信号较弱的地点

Wi‑Fi 版 iPad 比 iPhone 更容易成功，因为 iPhone 还会使用 GPS、蜂窝网络和基站信息判断真实位置。

## 三、准备香港 BSSID

可以参考以下开源研究工具：

- [SkyLift](https://github.com/adamhrv/skylift)
- [apple-corelocation-experiments](https://github.com/acheong08/apple-corelocation-experiments)

处理原则：

1. 查询香港某个固定坐标附近的 Wi‑Fi 数据。
2. 只保留合法的单播 BSSID。
3. 选择约 20～30 个彼此距离很近的 AP。
4. 使用苹果 WLOC 接口逐个复查。
5. 只使用仍能返回有效香港坐标的记录。

不要直接复制网上的旧 BSSID 列表。我们第一次测试的约 100 个地址中，绝大多数已经返回无效坐标，广播再多也没有用。

## 四、生成 ESP32 Beacon

ESP32 固件需要轮流广播这些香港 AP：

- 每个 Beacon 使用一个经过验证的香港 BSSID
- SSID 名称可以随意，例如 `HK-Test-01`
- 信道分散到 1、6、11
- 加入 802.11 Country Information Element

Country IE 参数：

```text
Country String: HK 
First Channel: 1
Number of Channels: 13
Maximum Transmit Power: 20 dBm
```

SSID 名称不是定位的关键。真正重要的是 BSSID 和 Country IE。

烧录时建议使用 `115200` 波特率。烧录完成后，从串口确认 Beacon 持续发送，并且发送失败计数为 0。

## 五、连接 iPad

1. 用 USB 将 iPad 连接到 Mac，并在 iPad 上选择“信任”。
2. 让 Mac 通过 USB 给 iPad 提供互联网连接。
3. 保持 iPad 的 Wi‑Fi 开启，使其能够扫描 ESP32 Beacon。
4. 将 iPad 和 ESP32 带到周围 Wi‑Fi、GPS 信号较弱的地方。
5. 启动 ESP32 Beacon 广播。
6. 在 Mac 上打开“控制台”，选择连接的 iPad。
7. 搜索 `countryd`、`wifid` 和 `RegulatoryDomain`。
8. 等待系统重新判断国家和监管域。

## 六、判断是否成功

成功时会看到类似日志：

```text
wifid: Updating CountryCodeFromWiFiAPs to HK
countryd: WiFi AP country code changed CN -> HK
countryd: location country code changed CN -> HK
RegulatoryDomain current estimates: HK
```

看到最终监管域为 `HK` 后，打开：

```text
设置 → AirPods → 听力健康 / 听力辅助
```

正常情况下，此时会出现听力测试和助听器设置。

系统可能存在约 600 秒的 Wi‑Fi 国家代码刷新保护。修改方案后不要连续重启或反复开关 Wi‑Fi，等待一段时间并以日志为准。

## 七、同步到其他设备

在 iPad 上完成助听器设置后：

1. 确认 Mac、iPhone 和 iPad 使用同一 Apple 账户。
2. 将同一副 AirPods 实际连接到目标设备。
3. 等待设置和资格状态同步。

我们的实测顺序是 iPad 先出现，Mac 随后出现，iPhone 最后出现。iPhone 可能存在缓存或同步延迟。

## 八、常见失败原因

### 只有 VPN，没有 Wi‑Fi 定位

公网 IP 已经是香港或美国，但旧的位置结果仍是中国，因此监管域不会改变。

### BSSID 已过期

公开列表中的 AP 可能已经移动、消失或被苹果数据库移除，必须逐个验证。

### 固件没有 Country IE

我们的第一版美国固件虽然使用了有效美国 BSSID，但没有写入 Country IE，最终监管域仍保持 `CN`。

### 只广播一个或几个 AP

少量 AP 很难形成稳定的位置判断。应使用同一小范围内的多个有效 BSSID。

### 简单使用铝箔

铝箔接缝和线缆会泄漏信号，普通包裹很难达到屏蔽箱效果。

### 直接使用 iPhone

iPhone 仍可能通过 GPS 和蜂窝网络得到真实位置。建议先用 Wi‑Fi 版 iPad 完成配置。

## 九、注意事项

- 苹果可能随时修改定位和监管域判断逻辑。
- BSSID 数据会过期，不能把某一批地址当作永久方案。
- “能看到测试 SSID”只说明 ESP32 正在广播，不能证明 iPad 已经采用这些 AP 定位。
- 最终应以 `countryd` 和 `RegulatoryDomain` 日志为准。
- 本文只记录一次成功实验，不代表苹果官方支持的开通方式。

更完整的原理和失败记录：

- [完整实验总结](https://gist.github.com/JerseyRiver/0aab824ef681cb76c9946bc2c23812bd)
