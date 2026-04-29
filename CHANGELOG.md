# Changelog

This tree started as stock `android12-5.10` GKI and then grew a lot of
small, opinionated tuning on top. What follows is roughly in the order
the changes happened — starting from the point DAMON landed, because
everything before that was just plain stock.

## Introduction of TCP Westwood

There are some real benefits TCP westwood bring against default bbr
so i figured out to keep it as =y in the defconfig so users can switch at will depending on their use case. Currently BBR is set as default, you can switch to westwood anytime.


## Bringing DAMON in

The first real move was documenting DAMON_RECLAIM. The code was
already in the tree, it just needed a proper opt-in entry in
`Documentation/admin-guide/pm/` so it wasn't a hidden knob. After
that, the Zenith governor got a `zenith.profile=` cmdline parameter
so the device could come up on a chosen preset (performance /
balanced / battery / legacy / custom) before userspace ever touched
anything.

Around the same time, PSI learned to track `MEMSTALL_RUNNING` without
growing `NR_PSI_TASK_COUNTS` — the original mainline fix widened a
kABI-visible counter, which would have broken GKI vendor modules. The
re-land here keeps the behavioural fix (reclaimers no longer count as
"full mem stall" when they're actually making progress) while leaving
the counter layout alone.

## Network and socket buffers

The default TCP congestion control got switched to Westwood+ for a
while, and `sysctl_optmem_max` was bumped from ~20 KiB to 64 KiB so
larger control-message footprints (epoll, netlink, multicast, 802.11
control frames) don't hit the socket option cap on real hardware.
Later on, the default TCP congestion control moved again, this time
to BBR — better single-stream throughput and noticeably less
bufferbloat on lossy mobile links than CUBIC.

## The DAMON backport 

The biggest chunk of commits in this section isn't mine — it's the
mainline DAMON subsystem as it stood around 6.x, cherry-picked onto
5.10. Everything from the DAMON operations refactor (targets by
index instead of id, `damon_primitives` → `damon_operations`,
paddr/vaddr registering themselves via `subsys_initcall`) through
the whole DAMON sysfs interface (schemes, quotas, prioritisation,
watermarks, stats, apply intervals), DAMON-based LRU-lists Sorting
(`LRU_PRIO` / `LRU_DEPRIO`), `nr_accesses_bp` and the pseudo-moving
sum, and piles of smaller fixes (divide-by-zero guards, memory-leak
fixes, KUnit coverage, allocation-failure handling in the sysfs tree).

These are all upstream work, cherry-picked clean. Credits at the
bottom of this file.

## VM defaults for a phone, not a server

The page allocator watermark scale factor went from 100 to 200, and
proactive compaction from 20 to 40. Neither is radical, but together
they give kswapd a tiny bit more headroom to reclaim before
allocations start failing under bursty app-launch loads, and
compaction runs slightly more often so order-4+ allocations don't
stall on a TLB-shattered zone during cold app starts. A bigger
"VM defaults for Android responsiveness and battery" pass went in
alongside, touching dirty ratios, swappiness, cache pressure — all
the usual suspects — to behave more like a phone and less like a
database box.

Per-bdi readahead later went from 128 KB to 256 KB. Sequential reads
off UFS/NVMe feel it — app loads, asset streaming, big file copies.

## BORE and the scheduler

BORE is the Burst-Oriented Response Enhancer scheduler that this tree
carries. Its defaults got tuned for Android UI smoothness in a single
pass first, then tightened in three more targeted passes:

- `sched_latency_ns` 24 ms → 16 ms (tighter scheduling period),
- `sched_min_granularity` 3 ms → 2 ms (shorter guaranteed timeslice,
  so latency-sensitive threads aren't starved as long),
- `sched_wakeup_granularity` 3 ms → 2 ms (a waking UI/input/binder
  thread can preempt a CPU-bound incumbent up to a full timeslice
  sooner).

`sched_nr_migrate` also came down from 32 to 8 — the default was
tuned for large x86 servers moving big batches of tasks; on an
eight-core phone, migrating eight at a time is plenty and keeps the
rq-lock hold short.

`CONFIG_HZ=300` got nuked in favour of a cleaner tick rate (and the
timer tuning that goes with it).

## Zenith, round after round

The Zenith governor itself is custom, and it got a lot of attention.
Ordered roughly as the commits landed:

`auto_tune` now ships enabled by default, so the governor actually
uses its own observer instead of sitting on whatever preset the
bootloader left. The defaults for `up_rate_limit` came down from
250 µs to 100 µs so the governor reconsiders going up 2.5× more
often; the up-spike bypass widened from 25% to 12.5% of max so short
bursts punch through the rate-limit floor at a lower threshold; and
`hispeed_load` came down from 90 to 65 so the snap-to-hispeed tier
actually engages on real interactive load instead of only near
saturation. A latent bug in the PERFORMANCE / BALANCED / BATTERY
presets where `hispeed_load` was set `>= up_threshold` — which meant
the hispeed step could never engage on those profiles — got fixed
next. Per-cluster `hispeed_freq` auto-defaults followed, so each
cluster picks its own snap-to frequency based on its own freq table
instead of inheriting one global number that fits neither big nor
little.

On the input-boost side, the boost ramp-down got smoothed so it
doesn't cliff-drop at the end of the window (the old behaviour would
hand back the cluster to PELT mid-gesture and you'd feel the frame
drop); and the boost learned to skip small clusters entirely by
default — touch UI and render run on big anyway, so dragging the
little cluster to its max on every tap was just wasted energy. More
recently: the boost full-pin phase now honours a per-profile ceiling
(`input_boost_cap_pct`) instead of always pinning to `policy->max` —
PERFORMANCE still pins to max, BALANCED caps at 80%, BATTERY at 60% —
and when a boost expires, the `sampling_down_factor` multiplier is
held for one extra down-rate window so PELT has a moment to catch up
before the limiter locks in a too-low choice.

`uclamp_max` became a symmetric final-freq cap so
PerformanceHint-style "prefer power efficiency" sessions are actually
capped at every decision tier, not just the entry one. The hispeed
floor tier gained hysteresis so it doesn't flap in/out when load
dances around its threshold. The per-policy uclamp aggregates now
cache for 1 ms, dropping the per-policy rq walk from every freq
decision to once per millisecond — pure CPU win, no behaviour
change. And the iowait boost now stacks on top of util instead of
replacing it, so mixed I/O + CPU workloads (loading a level while
shaders compile, app starts with DB reads and JIT threads running)
don't lose CPU headroom every time the disk wakes up.

Underneath all of that: PELT's halflife was halved via `multiplier=2`,
so the load tracker the scheduler and governor both read from reacts
twice as fast. Cold starts and bursty UI threads see the right
frequency sooner, and the cluster doesn't sit at a too-low choice for
30 ms after work arrives.

## Background noise, quietly turned down

A few small things the device shouldn't be spending cycles on during
normal operation:

- console loglevel dropped from 7 to 4 (no more normal-operation
  spam to serial / fb console);
- `vmstat stat_interval` went from 1 s to 10 s (background per-CPU
  stat work runs 10× less often; the accuracy hit only matters for
  debugging tools);
- `zram` default compressor switched from `lzo-rle` to `lz4` —
  significantly faster compress/decompress at almost the same ratio;
- power-efficient workqueues on by default (`WQ_POWER_EFFICIENT_DEFAULT=y`),
  so unbound work lands on efficient CPUs whenever that's an option.

## KMI: things tried, fixed, and reverted

Not every attempt stuck on the first try. A short detour here for
honesty:

`CONFIG_SCHEDSTATS=n` was the biggest single scheduler-path win —
it takes scheduler stat collection out of every context-switch path.
But it changes the size and layout of `struct sched_statistics`,
which is a real kABI break against stock GKI vendor modules. The
first pass disabled it outright; the second pass replaced that with
a layout-preserving shim so the perf win stays in-tree while vendor
modules still see the old struct layout and load. Same story on a
smaller scale with `CONFIG_PROFILING=n` and `CMA_DEBUGFS` / `CMA_SYSFS`
off — all reverted to keep the GKI story clean, because neither was
worth the vendor-module compatibility risk. `CONFIG_PERF_EVENTS=y`
got pinned explicitly so nothing later in the defconfig chain could
flip it off.

A couple of build fixes had to go in alongside the tuning: an unused
`megs` local in `mm/swap` that `-Werror=unused-variable` caught after
an earlier simplification, and a KVM perf stub that broke when
`CONFIG_PROFILING=n` briefly disabled the prototype it relied on.

## Storage scheduler, back to ADIOS

The `default_iosched` machinery had picked up some MQ-aware /
fua-handling experiments earlier in the tree. On the X6882's UFS
controller they turned out to be a net loss — more I/O latency
under bursty mixed read/write workloads than with the original
ADIOS path. So the whole MQ-aware / fua-aware experiment got
reverted in `U-i` and the elevator went back to plain `adios` as
the default. No regressions in `block/elevator.c`, no defconfig
churn — just a clean back-out of code that didn't earn its place.

## TEO refresh: one path that took four tries

The cpuidle TEO governor on android-5.10 was years behind upstream
when it came time to look at it. The first attempt (`U-g`) was a
22-commit incremental cherry-pick chain bringing TEO forward a
release at a time. That chain went sideways: each commit in the
middle relied on context that wasn't there yet, so `-X theirs`
conflict resolution silently kept old declarations alongside new
code. The build broke on `sched_cpu_util()` ABI mismatch (`U-g.1`
fixup), then on a missing `int i;` declaration in `teo_select()`
(`U-g.2` fixup), then on an undeclared local plus a duplicate
`end:` label (no fixup possible — the file had drifted past
hand-patching). At that point the whole TEO chain got reverted in
`U-g.3`, restoring `teo.c`, `cpuidle.rst`, `cpufreq_schedutil.c`
and `sched.h` to their pre-`U-g` state. The 8 non-TEO patches
inside the original `U-g` survived (mm `list_lru` follow-ups,
cpuidle don't-disable-on-suspend, sched-idle barrier, qspinlock
pending micro-opt, mm pageblock heuristic — all of which compiled
clean and don't touch the TEO code path).

After `U-g.3` cleared the deck, the build still failed on KMI
validation: a single `bool pending_event;` field that one of the
psi commits had added to `struct psi_trigger` changed the CRC of
`__traceiter_android_vh_psi_event` — a vendor tracepoint takes
that struct as its `TP_PROTO`, so layout change → CRC change →
KMI fail. The single offending commit got reverted in `U-g.4`;
the other psi fix (which only touches `kernel/sched/psi.c` body,
no struct change) stayed.

`U-k` then replaced TEO the way it should have been done in the
first place: a single drop-in patch lifting the production-tested
TEO from the millennium tree (which has been running this exact
file on 5.10 GKI for over a year). Util-awareness stays gated off
to match what millennium runs in production; no companion-file
changes; `cpuidle.rst` / `cpufreq_schedutil.c` / `sched.h` /
`defconfig` all untouched. The TEO upgrade we wanted out of `U-g`
landed clean as one file.

The lesson, which is now baked into the per-round pre-flight: for
every cherry-pick whose diff touches `include/linux/*` struct
fields, grep `include/trace/hooks/` for `TP_PROTO(struct $NAME *)`
and presume KMI break until proven clean. That check would have
caught both the original schedstats case and the `U-g.4` psi case.

## Small fixes, big lift each

`U-p` cherry-picks one upstream stable f2fs fix:
`cbc300d64926` ("f2fs: fix the missing write pointer correction").
Niche — fires only on zoned f2fs, which the X6882 likely doesn't
even use — but it's free safety against a real
checkpoint-disabled-mode bug, and it cherry-picks clean.

`U-j` cherry-picks two upstream zram fixes:
`13c1c74af764` (build warning fix when `ZRAM_WRITEBACK=n`) and
`212fe1c0df4a` (a real use-after-free in `zram_meta_alloc` failure
path: on alloc failure the code didn't NULL `zram->table`, so the
next `zram_meta_free` walked a dangling pointer; `Cc: stable`).
The originally-planned full zram recompression / multi-priority /
dictionary backport didn't happen — too much post-folio context
drift to bring back cleanly without the same kind of conflict
damage that broke `U-g`. Two commits, one file, real bug fixed,
no surface-area expansion.

A handful of survey rounds turned out to be no-ops on this tree:
EROFS picks (already in via 5.10.211 / 214 / 233 LTS merges),
sbitmap wakeup race fix `7610ba131925` (already in via 5.10.210),
schedutil rate-limit tune (Zenith already runs at `up=100`,
tighter than the proposed 500 µs). Saved three rounds of build +
test by checking what was already in place before cherry-picking.

## Zenith, three more feature packs

After all the cpuidle / KMI excursions, Zenith picked up three
back-to-back feature packs (`U-z1`, `U-z2`, `U-z3`). Each ships
its tunables **default-off** so the byte-for-byte behaviour
without userspace intervention is identical to the previous tree.
Userspace flips them on per-cluster as the device's display HAL,
init.rc, gameswitch helper, and similar daemons see fit.

### U-z1 — smoothness pack

Four pieces:

`predict_util_pct` (0..200, default 0) is a one-step-ahead linear
predictor that sits on top of `zenith_get_util()`. With a per-CPU
stored previous util and the current sample, the next freq decision
sees `util + (delta * predict_pct / 100)` instead of just `util`.
Up-only, so PELT still drives downsteps and the predictor never
pushes freq lower than reactive. Dampens the sawtooth pattern where
the governor undershoots a transient and then chases it for three
or four windows before catching up.

`render_aware` + `render_floor_pct` (default 0/70) walks the policy's
CPU `curr` tasks once every 4 ms (cached on the policy) and looks
for known render / display-pipeline `comm` strings:
`RenderThread`, `surfaceflinger`, `RenderEngine`, `mali-cmar-back`.
When one is on the cluster, freq is floored at `policy->max *
render_floor_pct / 100`. The 4-ms cache TTL means it costs at most
one CPU walk per 4 ms per policy — fine for 60 / 90 / 120 Hz panels.
Capped by uclamp_max downstream, so an explicit ADPF
power-efficiency hint still wins.

`game_mode` (0/1, default 0) is a single overlay flag. When
userspace flips it on (a small daemon watching
`/proc/touchpanel/game_switch_enable`, or the OEM gamemode service
if you have one), `hispeed_freq_pct` lifts by 10% and the
`input_boost_decay_ms` window stretches by 30%. No new tunable
chains, no behaviour-preset jungle — just a single boolean that
nudges two existing dials.

Three new tracepoints landed in the same patch:
`zenith_predict`, `zenith_render_floor`, `zenith_game_mode`. Plain
`TRACE_EVENT` definitions, no vendor-hook surface, KMI-clean by
construction.

### U-z2 — battery pack

Two pieces:

`psi_aware` + `psi_mem_thresh` (default 0/50) reads the system-wide
memory-pressure 10-second average from
`psi_system.avg[PSI_MEM_SOME][0]` (the same value
`/proc/pressure/memory` reports as `some avg10`). When the integer
percentage is at or above the threshold, the final freq is **capped
at the effective hispeed floor** — under heavy memstall, the
workload is memory-bound, so burning cycles above hispeed mostly
stalls waiting on memory rather than getting real work done. The
reader is RCU-free / lock-free: a single `READ_ONCE` is sufficient
because `psi_system.avg[][]` is written by a delayed-work
aggregator and 2 s of staleness is acceptable for a freq cap
decision. `CONFIG_PSI=n` and `psi_disabled` short-circuit cleanly.

`boot_boost_ms` (0..300_000, default 0) is a one-shot freq pin: for
the first N ms after `ktime_get_boottime_ns()` reads zero, freq is
pinned to `policy->max` (gated by `screen_state` and the existing
`input_boost_big_only` topology gate, so the small cluster doesn't
wake to max during boot). Past the deadline, the floor disappears
entirely — no per-call `ktime` read once the window has elapsed,
because the comparison short-circuits at the tunable check.
Recommended init.rc tune is `30000` (30 s); larger values are
accepted up to 5 min but rarely useful. Absorbs the cold-cache,
lots-of-zygote boot path without paying for the low-freq sample
windows that PELT would otherwise spend 200 ms catching up out of.

### U-z3 — adaptive frame budget

The "frame pacing" pack. The original plan was a real DRM vblank
hook (`drm_crtc_handle_vblank`-style callback into the governor),
but that path has KMI risk and MTK-specific glue overhead that
isn't worth the marginal gain over what `U-z1`'s `render_aware`
already buys. So `U-z3` ships a userspace-driven, lock-free,
KMI-zero alternative instead.

Two tunables: `frame_budget_us` (0..50000, default 0) and
`frame_pace_floor_pct` (0..100, default 0). A small userspace
daemon (10-line shell script that watches
`/sys/class/drm/card0-DSI-1/measured_fps` or `refresh_rate`, or
SurfaceFlinger if patched) writes the current vblank period in
microseconds whenever the panel changes refresh rate:
`16667` for 60 Hz, `11111` for 90, `8333` for 120, `6944` for 144.

The floor itself is computed adaptively. `frame_pace_floor_pct` is
the **60 Hz baseline**; the kernel scales it inversely with
`frame_budget_us` so shorter budgets need a higher floor without
userspace re-tuning:

```
eff_pct = (frame_pace_floor_pct * 16667) / frame_budget_us
floor   = policy->max * min(eff_pct, 100) / 100
```

So a baseline of 60% gives 60% at 60 Hz, ~90% at 90 Hz, 100% at
120 Hz and above. Sits between `render_floor` and `uclamp_max` in
the decision tier, so it stacks with the render-thread floor and
is still capped by ADPF power-efficiency hints.

A new tracepoint `zenith_frame_pace` fires once per call when
both tunables are non-zero, regardless of whether the floor moved
the freq — useful for sanity-checking that userspace is keeping
the budget in sync with vrefresh during live tuning.

This isn't real frame pacing — nothing in the kernel sees vblank
events directly, and there's no per-frame compute-deadline
awareness. But combined with `U-z1`'s `render_aware`, it covers
~90% of what a DRM-hooked frame pacer would deliver, with zero
KMI exposure.

### The full default-off feature stack

After `U-z1` + `U-z2` + `U-z3` the governor exposes seven
feature-flag tunables, all default-off:

| Tunable | Round | Purpose |
|---|---|---|
| `predict_util_pct` | U-z1 | One-step-ahead linear util predictor |
| `render_aware` + `render_floor_pct` | U-z1 | Render-thread comm-match floor |
| `game_mode` | U-z1 | Hispeed/decay overlay for games |
| `psi_aware` + `psi_mem_thresh` | U-z2 | Memstall-aware freq cap |
| `boot_boost_ms` | U-z2 | One-shot boot-window pin to `policy->max` |
| `frame_budget_us` + `frame_pace_floor_pct` | U-z3 | Adaptive vblank-paced floor |

Userspace flips them on per-cluster as needed. Defaults out of
the box are byte-identical to the pre-`U-z` tree, so anyone who
doesn't touch the new sysfs nodes sees no behavioural change at
all.

## What you actually feel

On a phone that boots this tree, the differences most people will
notice are: UI / touch / render threads wake up and get their CPU
slice faster (BORE + `sched_wakeup_granularity` + PELT halflife);
the governor picks a reasonable frequency on the first eval after
real work arrives, not 30 ms later (PELT + Zenith `hispeed_load` +
up-spike bypass + `predict_util_pct` once it's on); gestures don't
have a small dip at the tail (input-boost ramp-down +
`boost_exit_extend`); on devices that wire up `render_aware` and
`frame_budget_us`, the cluster stays at a render-friendly floor
through the whole gesture/animation rather than collapsing
between frames; battery behaviour under light load is slightly
better (per-profile boost cap + skip small clusters +
`powersave_bias` + `WQ_POWER_EFFICIENT_DEFAULT` + `psi_aware`
when on); the cpuidle path picks up the v6.x TEO improvements via
`U-k`; and the whole thing is quieter in background — less stat
collection, less log noise, less random kernel overhead per unit
of real work.

## Credits

The DAMON subsystem backport is the work of the upstream DAMON team.
The overwhelming majority of those commits is **SeongJae Park**, the
DAMON maintainer — everything from the operations refactor and the
sysfs interface through DAMON-based LRU sorting and the
`nr_accesses_bp` / pseudo-moving-sum work. Other upstream DAMON
contributors whose commits are in this tree:

- Pzqqt
- ramabondanp
- Baolin Wang
- Dan Carpenter
- Dawei Li
- Hyeongtak Ji
- Huan Yang
- Jinjie Ruan
- Jonghyeon Kim
- Kaixu Xia
- Kenneth Lee
- Miaohe Lin
- Xiaomeng Tong
- Xin Hao
- Xu Panda
- Yajun Deng
- Yu Zhe
- tangmeng
- tarsin

The BORE scheduler itself is upstream-adjacent work by Masahito
Suzuki and contributors — this tree just tunes its defaults; none
of the core BORE logic is mine. PELT, CFS, PSI, cpufreq infra,
schedutil / ondemand / reflex (which Zenith borrows heavily from) —
all mainline kernel work.

The TEO governor refresh in `U-k` is upstream work brought in as a
single drop-in from the `millennium-base` tree, which has been
running that exact file on 5.10 GKI in production for over a year.
The two zram fixes in `U-j` (`13c1c74af764`, `212fe1c0df4a`) are
upstream stable work by the zram maintainers; the f2fs fix in
`U-p` (`cbc300d64926`) is by Jaegeuk Kim. All cherry-picked clean
with original authorship preserved.

Other upstream contributors whose patches landed in the backport
window of this tree:

- Alex Rusuf, Chengming Zhou, Christophe JAILLET, Feng Tang,
  Gautam Menghani, Huaisheng Ye, Jianglei Nie, Jiapeng Chong,
  Kefeng Wang, Levi Yun, Mahdi Arghavani, Pzqqt,
  Rama Bondan Prakoso, Ryan Roberts, Su Hui, Yang Yingliang,
  andrew.yang.

The custom work — the Zenith cpufreq governor (including the
`U-z1` / `U-z2` / `U-z3` feature packs), the Android-oriented VM
and scheduler tuning, the KMI-safe PSI re-land, the defconfig
curation, the storage-scheduler ADIOS revert in `U-i`, and
everything else not attributed above — is by:

- **XTENSEI** (`xtensei.is.not.in.the.sudoers.file@protonmail.com`),
  also committing as `アドイ` on some earlier defconfig / network
  patches.

Thanks to everyone above, and to the Android Common Kernel and GKI
maintainers for keeping a workable 5.10 LTS base to build on.
