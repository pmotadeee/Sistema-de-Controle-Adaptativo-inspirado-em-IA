# 🧠 **Adaptive Control System — README**

This repository contains a prototype **Bayesian daemon** for adaptive optimization of Linux systems.
It dynamically adjusts:

* CPU governor
* TDP (Intel RAPL)
* Dynamic ZRAM configuration
* (Optional Turbo Boost — present in code but disabled)

The daemon runs in a continuous 5-second loop, gathering system metrics, updating a short-term memory state, classifying load levels, and applying policies accordingly.

Full documentation lives in:

* `README-DAEMON.md` — complete technical whitepaper
* `docs/daemon.md` — navigable docs
* `docs/diagrams.mmd` — Mermaid diagrams
* `Bansky/prototipos/proto-AGI/install.sh` — installer + systemd unit

This README provides a quick overview and safe testing workflow.

---

# 🚀 **Quick Start — Safe Testing**

## 1. **Inspect the installer before running**

```bash
sed -n '1,220p' Bansky/prototipos/proto-AGI/install.sh
```

## 2. **Test inside a VM (recommended)**

Example: Ubuntu VM

```bash
sudo apt update && sudo apt install -y lm-sensors util-linux zram-tools

sudo bash Bansky/prototipos/proto-AGI/install.sh

sudo systemctl status bayes_opt.service --no-pager
sudo journalctl -u bayes_opt.service -n 200 --no-pager
sudo tail -n 200 /var/log/bayes_mem/bayes.log
```

⚠ **Warning**
This daemon writes to `/sys`, loads/unloads kernel modules, and interacts with Intel RAPL + ZRAM.
Run in **VMs or isolated machines only**, not your production desktop.

## 3. **Run manually for debugging**

```bash
sudo /usr/local/bin/bayes_opt.sh &
tail -f /var/log/bayes_mem/bayes.log
```

If the binary doesn't exist, copy the script generated inside `install.sh` and run it locally before enabling it as a service.

---

# 📁 **File Structure**

The daemon maintains a small internal state directory:

```
/etc/bayes_mem/
    cpu_history           # rolling history (RNN-like memory)
    cpu_trend.log         # unused but available
    last_*                # persisted states (gov, tdp, zram)

/var/log/bayes_mem/
    bayes.log             # continuous log
```

---

# 🔌 **Main Execution Loop**

Main loop:

```
initialize_directories  
while true:
    log state
    apply_all
    sleep 5
```

Every 5 seconds the daemon:

1. Collects system metrics
2. Updates recurrent memory
3. Computes a discrete policy key
4. Applies CPU/TDP/ZRAM decisions
5. Logs everything

---

# 🧩 **1. Signal Collection (Sensors)**

### 🔥 CPU Usage

Reads `/proc/stat` twice, computes:

```
usage = (total_diff - idle_diff) / total_diff * 100
```

Equivalent to `top`/`htop` methodology.

### 🌡 Temperature

Extracted from `sensors` output.
Fallback: 40°C.

### 📊 Load averages

L1, L5, L15 via `uptime`.

### 📈 Load variance

`L1 - L5` — detects sudden load spikes.

---

# 🧠 **2. Recurrent Memory — faz_o_urro()**

Implements a minimal recurrent state:

1. Read history
2. Append new load value
3. Trim to `MAX_HISTORY=5`
4. Compute average
5. Persist to disk

This acts as a **degenerate RNN cell** using:

* short-term memory
* temporal pooling
* no gradients / no learning

Equivalent to an untrained GRU-like memory.

---

# 🔑 **3. Policy Mapping — determine_policy_key_from_avg()**

Converts load average → discrete performance profile:

```
000  
005  
020  
040  
060  
080  
100
```

Higher load = more aggressive profile.

---

# 🕹️ **4. Subsystems Controlled**

## A. **CPU Governor**

Low profiles → `powersave`
High profiles → `performance`

State persistence:

```
/etc/bayes_mem/last_gov
```

Cooldown:

```
gov_cooldown
```

Interface:

```
/sys/devices/system/cpu/cpufreq/policy*/scaling_governor
```

---

## B. **Turbo Boost (currently disabled)**

Would control:

```
/sys/devices/system/cpu/cpufreq/boost
```

---

## C. **TDP — Intel RAPL**

Each profile maps to:

```
MIN_W   MAX_W
```

Written to:

```
constraint_0_power_limit_uw
constraint_1_power_limit_uw
```

State persistence:

```
last_power
power_cooldown
```

---

## D. **Dynamic ZRAM Reconfiguration**

Affects:

* number of streams
* compression algorithm

Pipeline:

1. swapoff
2. rmmod zram
3. modprobe zram num_devices=N
4. assign algorithm
5. set disksize
6. mkswap + swapon

State persistence:

```
last_zram_streams
last_zram_algorithm
cooldown_zram
```

---

# 🧮 **5. Adaptive Cooldowns**

`calc_dynamic_cooldown()` considers:

* temperature
* load variance (L1-L5)
* subsystem “impact multipliers”:

  * governor: 1.0
  * turbo: 1.2
  * TDP: 1.5
  * ZRAM: 2.0

Prevents oscillation (“chattering”).

---

# 🔄 **5-Second Cycle Summary**

1. Read CPU usage
2. Update recurrent memory
3. Generate policy key
4. Log
5. Apply:

   * governor
   * TDP
   * ZRAM

---

# 🧬 **Why This Is a Degenerate Neural Network**

This daemon implements every component of a recurrent neural network — **except learning**:

| Component       | Real ANN      | This Daemon                   |
| --------------- | ------------- | ----------------------------- |
| Inputs          | sensor vector | CPU load, temp, variance      |
| Recurrent state | hₜ            | disk-based history average    |
| Weights         | W             | static thresholds/tables      |
| Activation      | f(x)          | governor/TDP/ZRAM adjustments |
| Forward pass    | x,h → y       | sensors → policy → action     |
| Backprop        | ✔             | ❌                             |
| Learning        | ✔             | ❌                             |

Therefore it behaves as:

> **A homeostatic recurrent agent (RNN) with fixed weights — a degenerate neural network.**

It performs:

* perception
* state update
* inference
* action

Just no gradient-based learning.

---

# 👑 **Technical Summary**

> A self-regulating adaptive daemon that collects runtime metrics, maintains a short-term recurrent state, classifies load levels into discrete policies, and applies system-level optimizations (governor, RAPL, ZRAM) with adaptive cooldown logic. Runs continuously as a systemd service.

---

# 🧩 **Recommended Next Steps**

* Always test in VM
* Add CI/CD + static analysis
* Add load simulator for policy tuning
* Consider a Python rewrite for better testability
* Convert this design into a full RNN (optional)

---

# 📎 **Links**

* Full docs: `README-DAEMON.md`
* Technical notes: `docs/daemon.md`
* Diagrams: `docs/diagrams.mmd`
* Installer: `Bansky/prototipos/proto-AGI/install.sh`
