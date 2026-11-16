# <img src="branding/app-icon.svg" alt="" width="40" height="40" align="absmiddle" /> Better Mi Fitness Sync (iOS)

把**小米运动健康（Mi Fitness）**的完整数据同步到 **Apple Health**：步数、心率、**完整睡眠分期（深睡/浅睡/REM/清醒）**、血氧、体重、运动记录（含 GPS 轨迹）等。

这是 [ilyasaftr/better-mi-fitness-sync](https://github.com/ilyasaftr/better-mi-fitness-sync) 的 iOS 定制版。原版要求**付费 Apple Developer 账号**才能写入健康数据，本版通过构建时注入 HealthKit entitlement，**免费 Apple ID + Impactor 即可安装使用**。

## 为什么需要它

小米运动健康官方 App 写入 Apple Health 的数据存在长期未修复的缺陷（大量用户报告）：

- **睡眠记录不完整**：仅写入一条「卧床」区间；若夜间出现清醒，其后的睡眠片段通常不再写入，导致 Apple Health 中出现「整夜仅睡数十分钟」的错误记录；
- **睡眠分期丢失**：深睡、浅睡、REM、清醒等分期数据不会写入，Apple Health 无法呈现睡眠结构，也没有任何睡眠质量分析；
- **其他数据写入不全**：心率、血氧、GPS 轨迹、历史数据的回填均不完整或缺失。

本应用直接从小米云读取与小米运动健康 App 内展示一致的完整数据，并按 Apple Health 的数据规范逐条写入，弥补官方同步的不足。

## 它能同步什么

| 数据 | 说明 |
|---|---|
| **睡眠** | 完整睡眠分期：深睡 / 浅睡 / REM / 清醒，逐段写入；另附全晚卧床样本。自动合并官方写入的残缺记录，用版本号覆盖 |
| **心率** | 按时间间隔的完整心率数据，可回填历史 |
| **血氧 SpO₂** | 按时间间隔的血氧记录 |
| **运动记录** | 每次运动的类型、时长、卡路里、心率区间，**含 GPS 轨迹**（从云端下载加密轨迹文件、本地解密解析后写入 Apple Health 的 Workout 路线） |
| **体重 / BMI** | 双向同步——写入 Apple Health，体重数据还能从 Apple Health 同步回小米云 |
| **血压** | 支持时自动写入 |
| **步数** | 按天的步数统计 |
| **其他** | 手环 App 里展示的其余指标按可用性同步 |

此外支持：**历史数据回填**（不限于最近几天，可拉取云端的全部历史）、**增量同步与去重**（稳定标识 + 版本号 upsert，重复同步不产生重复数据）、**定时自动同步**（可配置间隔）、登录凭据与健康数据全程只存本机。

## 安装（免费 Apple ID 即可）

1. 到 **Actions → build ipa** 下载最新构建产出的 IPA；
2. 电脑上安装 [Impactor](https://github.com/claration/impactor)（支持 Windows/macOS/Linux）；
3. iPhone 用 USB 连接电脑，Impactor 登录你的**个人 Apple ID（免费即可）**；
4. 拖入 IPA，勾选 **Only Register Main Bundle**，点 Install；
5. iPhone 上「设置 → 通用 → VPN 与设备管理」信任证书；
6. 打开 App → 登录小米账号 → 授权 Apple Health → 同步。

**不需要**付费开发者账号、不需要 Mac、不需要容器。

> 提示：免费 Apple ID 签名的 App 有效期 7 天，到期后用 Impactor 重签一次即可。

## 使用前提

- 手环数据需已通过**小米运动健康 App** 同步到小米云（本 App 不连接手环，只读云端数据）；
- 登录的小米账号需与小米运动健康中使用的账号一致；
- 建议在 Apple Health 中关闭小米运动健康的「写入」权限，避免两份数据冲突（保留其同步上云功能）。

## 构建

仓库自带 GitHub Actions 工作流 **`build ipa`**（`.github/workflows/build-ipa.yml`）：在 macOS runner 上编译 iOS IPA，并在打包前用 ad-hoc codesign 把 HealthKit entitlement 注入主二进制——这正是免费 Apple ID 能获得健康权限的关键。

手动触发：Actions 页面选择 `build ipa` → Run workflow。

本地构建：`bash ./gradlew :composeApp:linkReleaseFrameworkIosArm64` 后用 XcodeGen 生成工程并 `xcodebuild` 编译打包。

## 工作原理

- 使用 Kotlin Multiplatform（KMP）+ Compose，业务逻辑跨平台共享，iOS 外壳为 SwiftUI；
- 逆向小米运动健康云协议：小米账号登录获取 `serviceToken` 等凭据，RC4 变体签名算法请求 `*.hlth.io.mi.com` 加密接口；
- 按时间增量拉取步数/心率/血氧/睡眠/体重/运动等数据，GPS 轨迹经 FDS 预签名 URL 下载后 AES 解密解析；
- 通过 HealthKit 写入，以稳定的 syncIdentifier + syncVersion 做 upsert 去重，体重支持双向同步。

## 隐私

- 小米账号登录凭据仅保存在本机；
- 数据流向：小米云 → 你的 iPhone → Apple Health，不经过任何第三方服务器；
- 无数据采集、无遥测。
