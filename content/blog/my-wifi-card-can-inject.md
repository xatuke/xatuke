---
title: "I was certain my WiFi card couldn't inject packets. It could."
date: 2026-10-11T08:45:00+05:30
slug: ""
description: "A rtw89 card that 'can't' do packet injection, a confident wrong conclusion, and the regulatory footgun that was actually to blame."
keywords: ["linux", "rtw89", "monitor mode", "packet injection", "mac80211", "regulatory domain", "wireless"]
draft: false
tags: ["linux", "wireless"]
math: false
toc: true
---

As it can often be, I was poking at my laptop instead of doing anything useful. The question was simple: can my WiFi do monitor mode and packet injection? I was pretty sure it could. Famous last words.

What followed was an afternoon of me very confidently declaring my hardware broken, reading kernel source to prove it, and then discovering the real culprit was a one-line regulatory setting I'd nuked myself. So, a normal afternoon.

This is the whole thing, mistakes included.

> **Scope:** this is my own laptop, my own WiFi card, and my own home network. `aireplay-ng --test` only talks to access points I own. Don't point any of this at networks that aren't yours.

## The card

First, what am I even working with:

```bash
iw dev
```

```
phy#0
	Interface wlp16s0
		type managed
		channel 44 (5220 MHz), width: 80 MHz
```

So it's `wlp16s0`, not `wlan0` (systemd predictable names, thanks). The chip:

```bash
$ basename "$(readlink -f /sys/class/net/wlp16s0/device/driver)"
rtw89_8922ae
$ lspci | grep -i network
10:00.0 Network controller: Realtek RTL8922AE 802.11be PCIe Wireless Network Adapter
```

A brand new **Realtek RTL8922AE**, Wi-Fi 7, driven by the in-tree `rtw89` driver. If you've spent any time in this corner of Linux you already feel a slight sense of dread. Realtek + injection is not a happy phrase historically.

Monitor mode at least looked fine:

```
Supported interface modes:
	 * managed
	 * AP
	 * monitor      <-- there it is
	 * P2P-client
```

So monitor: advertised. Injection: let's find out.

## The confident wrong answer

The standard injection check is `aireplay-ng --test`. It needs the card in monitor mode on the right channel, which means kicking NetworkManager off the card first:

```bash
sudo airmon-ng check kill
sudo iw phy phy0 interface add mon0 type monitor
sudo ip link set mon0 up
sudo iw dev mon0 set channel 44
sudo aireplay-ng --test mon0
```

```
Trying directed probe requests...
30:CC:21:E3:50:24 - channel: 44 - 'Airtel_...'
 0/30:   0%
32:CC:21:D3:50:24 - channel: 44 - 'Airtel_..._5GHz'
 0/30:   0%
```

Zero out of thirty. Nothing came back. I did the responsible thing and double-checked by watching the TX counters while injecting:

```bash
# tx_packets before and after a run
0
0
```

Not a single transmitted frame. And no errors in `dmesg` either — the frames just evaporated. So I wrote the obituary: *monitor yes, injection no, classic rtw89, go buy an Atheros dongle.* I said it with my chest.

> **Note:** the TX counter thing comes back to bite me later. Remember that it read zero. _Epic foreshadowing._

## "no way it can work?"

The thing is, "the driver doesn't support it" and "I haven't made it work" are not the same sentence, and someone (fine, me, an hour later) kept poking at it. If injection were truly unsupported, *why* was it failing? Was my frame being rejected by the driver, by the firmware, or by something above all of that in the stack?

There's exactly one honest way to answer that: find where the frame dies. Time to read some code.

### Getting the *right* source

My first instinct — grab the out-of-tree `rtw89` backport and build it — fell over immediately:

```
error: implicit declaration of function 'init_dummy_netdev'
error: too many arguments to 'rtw89_ops_sta_rc_update'
```

The backport was too old for my kernel (7.1.8). Fighting that is a waste of time. The in-tree driver for *my exact kernel version* matches my running APIs by definition, so I pulled just that subtree straight from kernel.org:

```bash
curl -s https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.1.8.tar.xz \
  | tar -xJ --wildcards 'linux-7.1.8/drivers/net/wireless/realtek/rtw89/*'
```

### Following the frame

The TX entry point is `rtw89_ops_tx` → `rtw89_core_tx_write`. An injected management frame (a probe request is a mgmt frame) lands in `rtw89_core_tx_update_mgmt_info`, which conveniently has a debug print:

```c
rtw89_debug(rtwdev, RTW89_DBG_TXRX,
	    "tx mgmt frame with rate 0x%x on channel %d ...");
```

That means I don't even need to recompile to answer the "does the frame reach the driver" question — the driver already logs it, I just have to turn the log on:

```bash
echo 1 | sudo tee /sys/module/rtw89_core/parameters/debug_mask   # RTW89_DBG_TXRX
```

Inject again, then:

```bash
$ sudo dmesg | grep -c 'tx mgmt frame with rate'
0
```

Zero. The frame **never reaches the driver**. Whatever is eating it lives *above* rtw89, in `mac80211`. That actually flips the whole story — a driver can't be blamed for dropping a frame it was never handed.

### Where mac80211 drops it

Injected monitor frames go through `ieee80211_monitor_start_xmit()`. I pulled `net/mac80211/` from the same tarball and read it. Trimmed to the part that matters:

```c
// net/mac80211/tx.c, minus a lot of radiotap parsing
chanctx_conf = rcu_dereference(sdata->vif.bss_conf.chanctx_conf);
...
/* Frame injection is not allowed if beaconing is not allowed */
if (!cfg80211_reg_can_beacon(local->hw.wiphy, chandef, sdata->vif.type))
	goto fail_rcu;     // <-- our frame, right here
```

`cfg80211_reg_can_beacon()`. Injection is gated behind "are you even allowed to *transmit first* on this channel." And that is a **regulatory** question, not a hardware one.

## The actual villain

```bash
$ iw reg get
country 00: DFS-UNSET
	(5170 - 5250 @ 80), (6, 20), (N/A), AUTO-BW, PASSIVE-SCAN
$ iw phy phy0 info | grep 5220
	* 5220.0 MHz [44] (20.0 dBm) (no IR)
```

There it is in black and white. My regulatory domain is `country 00` — the "world" domain, the one the kernel falls back to when it has no idea where it is. And in the world domain, all of 5 GHz is marked **`no IR`**: *No Initiating Radiation*. You may listen, you may not speak first. No beacons, no active probes, no injection. mac80211 was dropping every single injected frame on channel 44 for a completely correct reason, several layers above the driver I'd been blaming.

Two things put me in the world domain, and I did both to myself:

1. **No regulatory database installed.** `wireless-regdb` wasn't present, so there was no `/usr/lib/firmware/regulatory.db` and the kernel had no country data to apply. Stuck on built-in world-`00`.
2. **`airmon-ng check kill`.** Normally NetworkManager sets my real country (India, `IN`) from the AP's beacon. I'd killed NetworkManager to enter monitor mode, so that never happened.

I had, with my own two hands, told the card it wasn't allowed to transmit, and then concluded the card couldn't transmit.

## Proving the hardware was innocent

Before fixing 5 GHz, the quickest sanity check: in the world domain, 2.4 GHz channels 1–11 are **not** `no IR`. If injection works there, the hardware is fine and this whole thing is purely regulatory.

```bash
sudo iw dev mon0 set channel 6
sudo aireplay-ng --test mon0
```

```
Injection is working!
...
30:15:77:88:CA:A2 - channel: 11 - 'Airtel_Superman'
 30/30: 100%
```

**`Injection is working!`** 30/30. The RTL8922AE injects perfectly. It had injected perfectly the entire time; I'd just been testing on a channel the regulator had padlocked.

> **Note:** remember the TX counter that read zero even when nothing was wrong? It *still* read zero here — on channels where injection demonstrably works at 100%. Turns out `tx_packets` on the netdev simply doesn't count injected monitor frames on this path. It was a red herring from the very first test. The real signal was the driver's mgmt-TX log line all along. I'd trusted the wrong number.

## Fixing 5 GHz

Install the regulatory database and set the country:

```bash
sudo pacman -S wireless-regdb
sudo iw reg set IN
```

```bash
$ iw reg get
country 00: DFS-UNSET       # ...still?
```

Still world-`00`. Of course. `cfg80211` reads `regulatory.db` exactly once, at init, via `request_firmware`. It had already tried and failed at boot (the file didn't exist yet) and cached that failure. Dropping the file in afterwards changes nothing; `iw reg set` has no data to work with. The kernel does have the signing keys built in (`CONFIG_CFG80211_USE_KERNEL_REGDB_KEYS=y`) and the db ships signed, so the *only* problem is that cfg80211 needs to be made to read it again.

Short of a reboot, that means reloading the wireless stack so cfg80211 re-runs `request_firmware` — which now succeeds:

```bash
sudo modprobe -r rtw89_8922ae rtw89_8922a rtw89_pci rtw89_core mac80211 cfg80211
sudo modprobe rtw89_8922ae
sudo iw reg set IN
```

```bash
$ iw reg get
country IN: DFS-UNSET
	(5150 - 5250 @ 80), (N/A, 30), (N/A)      # no more "no IR"
$ iw phy phy0 info | grep 5220
	* 5220.0 MHz [44] (30.0 dBm)              # clean
```

And the moment of truth, on channel 44, my own 5 GHz network:

```bash
sudo iw dev mon0 set channel 44
sudo aireplay-ng --test mon0
```

```
Injection is working!
32:CC:21:D3:50:24 - channel: 44 - 'Airtel_..._5GHz'
Ping (min/avg/max): 0.124ms/2.460ms/12.164ms Power: -27.47
 30/30: 100%
```

Done. Monitor *and* injection, 2.4 and 5 GHz, on a Realtek Wi-Fi 7 card with the stock in-tree driver. No patch, no firmware reverse engineering, no USB dongle. Just a card I'd falsely accused.

> **Note (a dumb one):** at one point `sudo` started rejecting my correct password. Not a typo — `pam_faillock` had locked me out (`deny=3`, `unlock_time=600`) because some earlier non-interactive `sudo` attempts had racked up three "failures." Ten minutes in the corner to think about what I'd done. If this happens to you: wait it out, or `faillock --user you --reset` from an already-root shell.

## Make it stick

The world-`00` trap only springs when the regdb is missing *and* something kills NetworkManager. `wireless-regdb` is installed now, so a normal boot self-heals. But for monitor-mode work (where you *do* kill NM), pin the domain so it survives:

```bash
echo 'options cfg80211 ieee80211_regdom=IN' | sudo tee /etc/modprobe.d/cfg80211.conf
```

After that, `IN` is applied at boot and the whole module-reload dance is never needed again.

### Enter monitor + injection mode

```bash
sudo sh -c '
  airmon-ng check kill
  iw reg set IN
  iw phy phy0 interface add mon0 type monitor
  ip link set mon0 up
  iw dev mon0 set channel 44   # your target channel
  echo READY
'
sudo aireplay-ng --test mon0
```

### Put everything back to normal

```bash
sudo sh -c '
  iw dev mon0 del
  ip link set wlp16s0 up
  systemctl restart wpa_supplicant NetworkManager
  echo 0 > /sys/module/rtw89_core/parameters/debug_mask
  echo RESTORED
'
```

## The lesson, such as it is

I burned an afternoon reading mac80211 source to prove a hardware limitation that didn't exist, when `iw reg get` would have told me the answer in the first thirty seconds. In my defense, the kernel dive is *exactly* what pointed me at `cfg80211_reg_can_beacon` and therefore at the regulatory domain — so it wasn't wasted, it was just the scenic route.

Two things I'll actually keep:

- **"The driver can't" and "I didn't let it" look identical from the outside.** Find where the frame dies before you blame anyone.
- **Don't trust a counter you haven't verified counts the thing you think it counts.** `tx_packets` read zero through this entire story and was lying the whole time.

Anyway — the card was fine. I was the bug. Thanks for the read, and bye!
