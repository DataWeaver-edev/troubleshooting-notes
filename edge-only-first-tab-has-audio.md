# Only the first tab has sound; new tabs are silent — fix: per-app audio Output must be "Default" (blank ≠ Default)

> Search keywords: browser first tab has audio but new tab silent / Edge only one audio session /
> Edge multi-tab only one tab plays sound / 第一个标签页有声音 新标签页没声音 /
> Edge 多标签只有一个有声音 / Edge 无声音 重启后只有第一个标签页

| | |
|---|---|
| **Product** | Microsoft Edge 155.0.4283.60 (Stable, 64-bit) |
| **OS** | Windows 10 Pro 22H2, build 19045 |
| **Board** | GIGABYTE Z370 AORUS ULTRA GAMING 2.0 (rev 1.0) |
| **Audio** | Realtek ALC1220, driver 6.0.9313.1 (DCH / UAD) · NVIDIA HDMI/DP audio also present |
| **Status** | ✅ Solved |
| **Date** | 2026-10-16 |

## TL;DR

In **Settings → System → Sound → Advanced sound options → App volume and device preferences**,
Microsoft Edge's **Output** was **blank**. Setting it explicitly to **Default** restored audio on
every tab instantly.

**Blank is NOT the same as Default.**

- `Default` = explicitly follow the system default output device
- `Blank` = no effective value, so Windows falls back to a *stale per-app binding*

## Symptom

1. Launch Edge. The **first tab** that plays audio is audible.
2. Every **newly opened tab** that plays audio is **completely silent**.
3. The media really is playing: the progress bar advances, no error, no buffering stall — just no sound.
4. Volume Mixer shows only **one** audio session for Edge (not muted, volume normal).
5. Extra: pause a playing tab for ~5 s then resume → also silent. A short pause/resume still has sound.
6. In an InPrivate window even the **first** tab is silent.
7. Another Chromium-based browser on the same machine plays audio on its second tab normally.

## Environment

- Microsoft Edge 155.0.4283.60 (Stable, 64-bit)
- Windows 10 Pro 22H2, OS build 19045
- Motherboard: GIGABYTE Z370 AORUS ULTRA GAMING 2.0 (rev 1.0)
- Onboard audio: Realtek ALC1220, driver 6.0.9313.1 (DCH / UAD)
- Other endpoints present: NVIDIA HDMI / DisplayPort audio
- Output device in use: front-panel headphone jack

## Root cause

Windows stores a per-application audio endpoint binding in:

```
HKCU\SOFTWARE\Microsoft\Internet Explorer\LowRegistry\Audio\PolicyConfig\PropertyStore
```

Two stale entries for `msedge.exe` were still present:

```
[1158b833_0] ...\rtspdiftopo/00010001|...\msedge.exe   ->  Realtek ALC1220 S/PDIF
[fcb54032_0] ...\topo00/00010001|...\msedge.exe        ->  NVIDIA HDMI
```

Both endpoints have nothing physically attached. In the Settings UI, Edge's **Output** dropdown
showed **blank** — not "Default", and not any device. Blank means "no effective value", so Windows
fell back to the residual binding. The first audio session grabbed the dead endpoint; sessions
created afterwards got nothing.

Full registry entries:

```
[1158b833_0]
  {2}.\\?\hdaudio#func_01&ven_10ec&dev_1220&subsys_1458a182&rev_1000
   #{6994ad04-93ef-11d0-a3cc-00a0c9223196}\rtspdiftopo/00010001
  |...\Microsoft\Edge\Application\msedge.exe

[fcb54032_0]
  {2}.\\?\hdaudio#func_01&ven_10de&dev_0099&subsys_1b4c1366&rev_1001
   #{6994ad04-93ef-11d0-a3cc-00a0c9223196}\topo00/00010001
  |...\Microsoft\Edge\Application\msedge.exe
```

## Fix

1. Settings → System → Sound → Advanced sound options → App volume and device preferences
2. Find **Microsoft Edge**
3. Set **Output** to **Default** (it was blank)
4. No reboot, no browser restart — audio comes back immediately

## Ruled out (none of these helped)

- Exclusive mode / "Allow applications to take exclusive control of this device" — on or off
- Realtek Console multi-stream mode
- Edge hardware acceleration — on or off
- `Ctrl+M` tab mute (not muted)
- `--disable-features=AudioServiceOutOfProcess`
- In-player site settings (a video platform's "volume balance" / audio normalization)
- Changing the audio source or site (reproduced on two independent online-music sites)
- Updating the Realtek driver (6.0.1.8470 → 6.0.9313.1)
- Fresh Edge profile via `--user-data-dir`

## Expected behaviour

- The App volume and device preferences UI should **never render an empty Output**. If there is no
  explicit binding, show "Default".
- The browser should not silently route audio to a stale endpoint with no device attached; fall back
  to the system default instead of going silent with no error.
- Some visible indication that the output endpoint is unavailable would have saved several hours.

## 中文摘要

**现象**：Edge 启动后第一个标签页有声音，之后新开的标签页全部静音；视频正常播放、进度条在走，
只是没声音；音量合成器里 Edge 只有一个会话；无痕窗口连第一个标签页都没声；
同机另一款 Chromium 浏览器第二个标签页正常。

**根因**：Windows「按应用」的音频设备绑定残留。注册表
`HKCU\SOFTWARE\Microsoft\Internet Explorer\LowRegistry\Audio\PolicyConfig\PropertyStore`
中 `msedge.exe` 仍绑定在 Realtek S/PDIF 与 NVIDIA HDMI 两个未接设备的端点上；
而「设置 → 系统 → 声音 → 高级声音选项 → 应用音量和设备首选项」里 Edge 的「输出」显示为**空白**。
空白不等于默认，系统因此回落到那条残留绑定。

**解决**：把 Edge 的「输出」从「空白」显式改成「**默认**」，立刻恢复，无需重启。

**关键区分**：**空白 ≠ 默认**。

---

Cross-posted to
[Microsoft Tech Community (Edge Insider → Discussions)](https://techcommunity.microsoft.com/discussions/edgeinsiderdiscussions/only-the-first-tab-has-sound-new-tabs-silent---set-per-app-output-to-default-bla/4565129)
so other users — and their troubleshooting agents — can find this by symptom.
