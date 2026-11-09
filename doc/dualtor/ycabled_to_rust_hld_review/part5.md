<!-- Part 5 of 5 of doc/dualtor/ycabled_to_rust_hld.md (sonic-net/SONiC#2635), lines 1094-1293. Split only because the file exceeds Copilot's per-file review limit; links to sections in other parts do not resolve here. -->

## 5 Performance and memory

### 5.1 Expected results

The baselines are in [Section 1.2.2](#122-resource-usage). The active-active numbers on a DUT and all numbers on physical testbeds are still to be collected.

| Metric | Python | Rust target | Why |
| --- | --- | --- | --- |
| Healthy-port probe or toggle latency with K unreachable SoCs | Up to K × 0.5 s | Independent of K | Per-port actors |
| Startup, N active-active ports, SoCs unreachable | N × 0.5 s plus channel setup | About 0.5 s | Parallel initialization |
| Telemetry pass, 32 unreachable SoCs | 64 s, longer than the 60 s period | About 2 s per port, in parallel | Per-port timers |
| RSS, active-active only | 41 MB (dev VM, 32 channels) | ≤ 15 MB | No interpreter |
| RSS, active-standby | 53-54 MB (KVM, 24 ports) | ≤ 37 MB (70%) | Interpreter only for the drivers, no per-thread DB connectors |
| Threads | 5, plus about 1 per active-active channel (53 with 32 channels) | ≤ 16, plus one per driver call in flight | tokio; blocking threads only while a driver call runs |
| Redis connections | 124 (KVM, 24 ports) | ≤ 48 per namespace (4 by design) | One connection per database and one pub/sub connection for all subscriptions (D9) |
| Idle CPU | 0.7-0.8% of one core (KVM, 24 active-standby ports) | No higher than Python's | Event-driven, no 100 ms select loop |

### 5.2 How to measure

Run both implementations on the same DUT and image, switching with `use_rust_ycabled`:

- **Memory:** RSS and PSS from `/proc/<pid>/smaps_rollup`, threads from `/proc/<pid>/status`, Redis connections with `ss -tanp`. Measure 10 minutes after startup and again after a toggle storm, and record the peak thread count during startup and the storm.
- **Latency:**
  - toggles: from orchagent's write of the request to ycabled's STATE_DB `HW_MUX_CABLE_TABLE` write, which includes the time a request waits behind others; `MUX_METRICS_TABLE` `xcvrd_switch_*` cover only the handling
  - probes: time from the command write to the response write, in the test harness
- **Fault injection:** make K SoCs unreachable with nic_simulator or by dropping their traffic with iptables.
- **Startup:** from supervisord's "spawned" message to the last `HW_MUX_CABLE_TABLE` row written.
- **CPU:** utime plus stime over one idle hour, and during telemetry passes.

### 5.3 Measured results

`sonic-ycabled/blackbox/bench` ran both implementations one after the other, in the black-box suite's container (docker-sonic-vs, Debian 12, Python 3.11) on a shared 16-core host. Both ran on the suite's fakes: a vendor driver that forwards every call to a cable model in the test process over a Unix socket, and gRPC SoC doubles in the test process. So the numbers compare the daemons, not hardware: real drivers' I2C calls are slower, and numbers on KVM and physical testbeds are still to be collected.

| Scenario | Metric | Python | Rust |
| --- | --- | ---: | ---: |
| 24 active-standby ports | RSS / PSS | 50.3 / 42.0 MB | 38.3 / 33.8 MB |
| | Threads | 5 | 3 |
| | Redis connections | 125 | 4 |
| | CPU over 65 s, with one telemetry pass | 0.14% | 0.14% |
| | Startup, until the suite sees every subscription | 0.88 s | 0.73 s |
| 32 active-active ports, one SoC each | RSS / PSS | 58.8 / 48.6 MB | 12.1 / 10.6 MB |
| | Threads | 61 | 3 |
| | Redis connections | 199 | 4 |
| | CPU over 65 s, with one telemetry pass | 4.31% | 0.14% |
| | Startup, until the suite sees every subscription | 0.86 s | 0.41 s |
| Startup, SoCs accept TCP and never answer: first DB access to every port's first state | 8 ports | 4.16 s | 0.51 s |
| | 32 ports | 16.30 s | 0.52 s |
| 8 active-active ports toggled at once, the SoCs of 4 hang | Healthy ports, slowest answer | 2021 ms | 6.9 ms |
| | Hanging ports, slowest answer | 2014 ms | 502 ms |
| Active-standby probe, 200 in a row | Median / p99 | 1.65 / 2.12 ms | 1.88 / 2.55 ms |
| 24 active-standby ports toggled together, 10 rounds | Toggles per second | 936 | 1063 |
| First telemetry pass of 32 active-active ports, first write to last | SoCs connected, telemetry RPCs hang | 62.3 s | 2.0 s |
| | SoCs blackholed | 0.04 s | 2.0 s |

- **Memory:** an active-active device runs no Python, so ycabled-rs needs a fifth of Python's memory. With active-standby ports, the embedded interpreter and the driver modules account for most of it: 38.3 MB is 76% of Python's 50.3 MB, short of the 70% target of [Section 5.1](#51-expected-results). Every other target is met.
- **Threads:** the runtime's 2 workers and the main thread. Threads for driver calls come and go. Python's count grows with the active-active ports: grpcio polls each channel's connectivity in a thread of its own.
- **CPU:** the active-active difference is grpcio's polling threads. With active-standby ports, both are idle.
- **Latency:** Python's single request loop makes the healthy ports wait for the 4 hanging ones' 0.5 s deadlines, and initializes ports one after another. Probe latency is the fake cable's round trip for both.
- **Telemetry:** with hanging SoCs, Python's serial pass (4 RPCs of 0.5 s per port) takes longer than the 60 s period. Blackholed SoCs are the exception: once grpc-core's first connection attempt has failed, it fails RPCs at once, and ycabled-rs waits for each deadline, 16 ports at a time ([Section 4.3.4](#434-soc-grpc-client)). The pass runs beside the actors, so no request waits for it.

## 6 Testing

### 6.1 Rust unit tests

142 tests, in `sonic-ycabled-rs`. Most run a port actor against mocks of the DB, the platform and cables, and the SoC (`src/testing.rs`). Behavior at a boundary is tested against the real component: a private `redis-server`, an in-process gRPC server, and the embedded interpreter with fake drivers. Among the cases:

- read_side mapping
- reply parsing
- the timestamp format and Python `str()` formatting
- the connectivity state machine
- subscriptions: the existing rows first, then the changes, in the order redis delivered them across tables
- mailbox ordering, toggles beside a telemetry pass, and a port held by a request beside the other ports' passes
- every CLI command family, with its failures

Each test names the Python unit tests whose behavior it covers, and `sonic-ycabled-rs/doc/python-ut-coverage.md` maps all 284 Python tests: covered, or why not (Python's threads, breakout ports, functions nothing calls).

Line coverage must be at least 80% (N11). The tests run with `cargo test`; the package build and the sonic-platform-daemons CI will run them once they build the crate.

### 6.2 Parity tests

Until the harness of [#915](https://github.com/sonic-net/sonic-platform-daemons/pull/915) exists, the black-box suite ([Section 6.3](#63-black-box-component-tests)) is the parity test: it runs either daemon on the same inputs, with the same assertions and the same DB contract.

- **Harness:** the pmon-common parity harness ([#915](https://github.com/sonic-net/sonic-platform-daemons/pull/915)) runs the Python daemon and the Rust daemon on the same inputs and compares what they write and log.
- **Inputs for ycabled:**
  - a real Redis with CONFIG_DB, APPL_DB and STATE_DB contents
  - a scriptable SoC double serving both gRPC services
  - a scriptable Python Y-cable driver
  - a fake platform for presence and transceiver info
- **Cases:** the scenarios of the existing Python unit tests, plus the cases in [Section 6.3](#63-black-box-component-tests).
- **When Python changes:** a parity test fails, so the two implementations cannot drift apart unnoticed.

### 6.3 Black-box component tests

`sonic-ycabled/blackbox` in sonic-platform-daemons ([#941](https://github.com/sonic-net/sonic-platform-daemons/pull/941)) runs the daemon as a process against a real redis, in a throwaway container: one built on docker-sonic-vs (`run.sh`), or the sonic-slave container of the sonic-platform-daemons pipeline, where `pytest` runs them after ycabled's unit tests. The tests see only what the daemon writes to the databases, sends to the SoCs, asks of the cables and logs. `BB_YCABLED_CMD` runs another implementation with the same inputs.

- **Stand-ins:**
  - orchagent, linkmgrd, xcvrd and the `show/config muxcable` CLI write the requests, with the CLI's own table protocol.
  - A fake vendor driver is installed in a copy of `sonic_y_cable` and mapped like a real one. It and a fake `sonic_platform` forward every call to a cable model in the test, which can return values, raise, delay or block. The suite also records any call that starts while another call to the same cable is still running.
  - gRPC SoC doubles serve both services from the real proto, with TLS when asked.
  - A mux simulator serves the real `y_cable_simulated` driver on VS.
  - A syslog sink on `/dev/log` records the logs.
- **Checks on every test:** redis MONITOR records every command of the daemon. A test fails if the daemon writes a table, field or value outside a machine-readable DB contract (200 entries, from [Section 3.5](#35-db-schema)), if it logs a crash, or if it exits unexpectedly.
- **Known issues:** each issue in [Section 3.7](#37-known-issues) that a test can observe has a test that asserts the correct behavior; Section 3.7 names the tests. These tests are strict xfails on Python ycabled, so they flag a fix in Python, and must pass on Rust. N1 and N3 have such tests too. Each test explains its issue in its `known_issue` reason and docstring, so the suite does not depend on this document. Only an `AssertionError` counts as the expected failure: the harness raises a `RuntimeError` when it cannot do what a test asks, and every test has a time limit (300 s), so a harness error, a timeout or a hang cannot pass as the issue.
- **Results on Python ycabled:** 307 tests pass and 29 are expected failures (27 for 25 known issues, N1, N3), in about 14 minutes.
  - The tests exercise all 200 DB contract entries. A run of the whole suite on Python ycabled fails if one is not exercised, so the contract cannot keep entries no test reaches or accesses ycabled no longer makes.
  - They cover 87% of ycabled's lines and branches (90% of its statements). The unit tests reach 81%.
  - Of the 342 statements not reached: 70 are the shutdown path, which issue 1 blocks; 82 are dead code, unused getters and import fallbacks; most of the rest are guards no input reaches, such as breakout ports, unknown event codes and `select` errors.
- **Results on ycabled-rs:** 334 tests pass, the known-issue tests included, and the 2 tests of Python-only mechanisms are skipped, in about 10 minutes. The run exercises all 200 DB contract entries.
- **In the pipeline:** the ycabled test step sets `BB_ENABLE=1`, which lets the tests take over the container, and runs them after the unit tests. The step takes about 14 minutes instead of 15 seconds. The ycabled coverage it reports, which diff coverage uses, goes from 81% to 95% of statements (N12).

| Area | Cases |
| --- | --- |
| Lifecycle | Startup with active-standby, active-active and mixed ports; no mux ports; waiting for `CONFIG_DB_INITIALIZED`; port map from CONFIG_DB `PORT` or `port_config.ini`; startup failures; SIGTERM, SIGHUP; restart and replay of the last request; task failure |
| Insertion and removal | `TRANSCEIVER_INFO` SET and DEL; SET for an initialized port ([#882](https://github.com/sonic-net/sonic-platform-daemons/pull/882)) and for a driver that is not initialized; absent at startup, inserted later |
| Active-standby | Toggle to active and standby for each read_side; toggle failures; read side or mux direction unreadable; probe; every driver-load failure, including `get_vendor()` mismatch and exceptions; simulator override on VS |
| Active-active | Self and peer toggles; forwarding state probe; status codes, malformed replies, hangs; no, invalid or IPv6 SoC address; absent cable; SoC up later or lost (`failure`); insecure, server-auth and mutual TLS and their misconfigurations; GracefulRestart streams; N1, N3 |
| Telemetry | Both cable types and read sides; driver values and exceptions; absent cable, no driver, unreachable SoC; download in progress; telemetry disabled and enabled again |
| CLI | Every command family in [Section 3.5.5](#355-cli-command-tables) with its arguments, driver failures, missing arguments, missing driver, cable gone and malformed commands; firmware download in progress |
| Configuration | `XCVRD_LOG` knobs, `grpc_secrets.json`, Loopback3 variants, VS platform, log texts sonic-mgmt matches, Redis connection count, multi-ASIC namespaces |
| Concurrency | A toggle or an insertion during another call to the same cable; a removal during a toggle; a toggle during an insertion, or before the driver is verified; calls after a removal |

### 6.4 System tests

- **Acceptance:** on the physical dualtor testbeds Libra and Gemini, which cover both cable types, run the full dualtor suite with Rust ycabled and compare with a Python baseline on the same image:
  - `tests/dualtor`, `tests/dualtor_io`, `tests/dualtor_mgmt`
  - the dualtor cases of `arp`, `decap` and `qos` (`test_tunnel_qos_remap`)
- **Log analyzer:** enabled, so new ERR messages fail a test.
- **KVM, with each implementation:**
  - active-standby: the `dualtor` set of sonic-mgmt's `pr_test_scripts.yaml`, which sonic-buildimage's PR check `kvmtest-dualtor-t0` runs on the `dualtor` topology with the mux simulator
  - active-active, which has no upstream KVM PR check: `vms-kvm-dual-vpp-t0-2` (`dualtor-aa-vpp`, with nic_simulator) or `vms-kvm-dual-mixed` (`dualtor-mixed`)
- **By hand:** CLI commands that no suite runs, such as firmware download, activate and rollback with an image the vendor has approved, and PRBS and loopback.

### 6.5 Performance, fault and soak tests

- The measurements in [Section 5.2](#52-how-to-measure), including a toggle storm on all ports and K unreachable SoCs.
- A 72-hour soak with periodic toggles, SoC restarts, cable removal and insertion where the testbed allows it, pmon restarts and config reloads.

### 6.6 Acceptance criteria

- All unit, parity and black-box tests pass. The black-box tests pass on both implementations.
- On each physical testbed, the dualtor suite results with Rust match the Python baseline over 3 runs, and every difference has a known cause. The log analyzer reports nothing new.
- The targets in [Section 2.3](#23-non-functional) are met, or the reviewers accept the difference.
- The soak run has no restarts, no growth in memory and no port stuck in `unknown`.

## 7 Rollout and migration

| Phase | What | Exit criteria |
| --- | --- | --- |
| 0 | Merge the crate, the package and the supervisord switch, with `use_rust_ycabled` off everywhere | CI passes; the image builds on all platforms |
| 1 | KVM dualtor tests with both implementations ([Section 6.4](#64-system-tests)); acceptance on the physical testbeds | [Section 6.6](#66-acceptance-criteria) |
| 2 | Turn on in the nightly dualtor test plans | 4 weeks without regressions caused by ycabled |
| 3 | Turn on by default for the dualtor HwSKUs, in their device files | Release sign-off |
| 4 | Retire Python ycabled, one release after phase 3 | Agreement of the owners |

Changes to Python ycabled during phases 0-3 must be made in both implementations. The parity tests enforce this. At any phase, removing the key and restarting pmon goes back to Python.

## 8 Risks and mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| The shared framework ([#915](https://github.com/sonic-net/sonic-platform-daemons/pull/915), [#29860](https://github.com/sonic-net/sonic-buildimage/pull/29860)) changes or merges late | Rework, or a blocked merge | Keep ycabled's dependency on pmon-common narrow. If needed, build in a workspace of its own as [#936](https://github.com/sonic-net/sonic-platform-daemons/pull/936) does, and move later. |
| tonic's connections behave differently from grpc-core's connectivity states | linkmgrd sees `failure` at different times | Connector wrapper and state machine ([Section 4.3.4](#434-soc-grpc-client)); parity tests with a fault-injecting SoC double; fall back to the grpcio crate if the gap cannot be closed |
| Embedded Python: the GIL, drivers that start threads or call `openlog`, signal handling | Hangs, or a changed log identity | Interpreter without signal handlers; logging owned by the bridge; a worker pool and a watchdog |
| Python ycabled keeps changing | The Rust version falls behind | Parity tests in CI; changes made in both until Python is retired; freeze Python before phase 3 |
| `TRANSCEIVER_INFO` as the active-active presence source (D3) | Different behavior while xcvrd has not posted yet | Insertion events are already handled; parity test with a late `TRANSCEIVER_INFO`; fall back to the platform API |
| Access to physical testbeds and cables from several vendors | Acceptance takes longer | Book the testbeds early; test day to day on the KVM testbeds ([Section 6.4](#64-system-tests)) |
| Review bandwidth: the Rust pmon PRs have been open since September 2026 | Slow merges | Small PRs, one per component; coordinate with the owners of the Rust pmon daemons |

## 9 Open questions

1. Is STATE_DB `TRANSCEIVER_INFO` acceptable as the presence source for active-active ports (D3), or must presence come from the platform API?
2. Should vendor drivers be reached through a ycabled-local bridge, or through a Y-cable facade added to sonic-platform-common's generated platform API ([#768](https://github.com/sonic-net/sonic-platform-common/pull/768))?
3. gRPC: tonic with derived connectivity states (D5), or the grpcio crate, which uses grpc-core? Its semantics would be closer to Python's, though not identical: its last release (0.13.0, August 2023) has C core 1.56, and Python ycabled requires grpcio 1.71.
4. Do the maintainers agree to fix the Python issues in [Section 3.7](#37-known-issues) before the port, so both implementations share the corrected behavior?
5. `GRPCCLIENT|config.auth_level`: should YANG (`server`, `client`) or the code (`server`, `mutual`) change?
6. Is keeping the tables on stop (C7), which is what Python does in practice, the intended behavior, rather than deleting them as `deinit()` was written to do?
7. Is `pmon_daemon_control.json` enough to select the implementation, or do labs need an override that does not edit device files?
8. When should Python ycabled be retired after Rust becomes the default?
9. GracefulRestart `tor`: Python sends `UPPER_TOR` from the lower ToR (read_side 0) and `LOWER_TOR` from the upper ToR. That is the opposite of the ToR names and of the enum's values, which follow the port ids (`LOWER_TOR = 0`, `UPPER_TOR = 1`). The proto does not define the sides, and sonic-mgmt's NIC simulator does not implement the service. Which side does the SoC expect? Until its owners confirm, ycabled-rs sends what Python sends.
10. Is inserting or removing a cable, or changing `MUX_CABLE`, after ycabled starts a supported scenario? Python ycabled has a thread for cable insertion and removal, but the issues in [Section 3.7.5](#375-cable-insertion-and-removal-after-startup) show that this path is incomplete. If the procedure for replacing a cable restarts ycabled instead, those issues need no Python fix; 18 and 24 still do, for startup. ycabled-rs handles these changes either way.

## 10 References

- [Active-Active Dual ToR HLD](active_active_hld.md) and [Active-Standby Dual ToR HLD](dualtor_active_standby_hld.md)
- Python ycabled: [sonic-platform-daemons/sonic-ycabled](https://github.com/sonic-net/sonic-platform-daemons/tree/master/sonic-ycabled)
- This design: black-box tests [sonic-platform-daemons#941](https://github.com/sonic-net/sonic-platform-daemons/pull/941), Rust implementation [sonic-platform-daemons#944](https://github.com/sonic-net/sonic-platform-daemons/pull/944)
- Rust pmon daemons:
  - [sonic-platform-daemons#915](https://github.com/sonic-net/sonic-platform-daemons/pull/915): workspace, pmon-common, thermalctld
  - [#916](https://github.com/sonic-net/sonic-platform-daemons/pull/916) to [#920](https://github.com/sonic-net/sonic-platform-daemons/pull/920): the other daemons
  - [#936](https://github.com/sonic-net/sonic-platform-daemons/pull/936): xcvrd
- Platform API facade: [sonic-platform-common#768](https://github.com/sonic-net/sonic-platform-common/pull/768)
- Build and selection: [sonic-buildimage#29860](https://github.com/sonic-net/sonic-buildimage/pull/29860)
- Vendor drivers: [sonic-platform-common/sonic_y_cable](https://github.com/sonic-net/sonic-platform-common/tree/master/sonic_y_cable)
- linkmgrd: [sonic-linkmgrd](https://github.com/sonic-net/sonic-linkmgrd) `DbInterface.cpp`, `MuxPort.cpp`, `LinkManagerStateMachineActiveActive.cpp`
- orchagent: [sonic-swss](https://github.com/sonic-net/sonic-swss) `orchagent/muxorch.cpp`
- CLI: [sonic-utilities](https://github.com/sonic-net/sonic-utilities) `show/muxcable.py`, `config/muxcable.py`
- Rust swss-common: [sonic-swss-common/crates/swss-common](https://github.com/sonic-net/sonic-swss-common/tree/master/crates/swss-common)
- sonic-mgmt: `tests/dualtor*`, `ansible/roles/test/files/tools/loganalyzer/loganalyzer_common_ignore.txt`
- [tonic](https://github.com/hyperium/tonic), [PyO3](https://github.com/PyO3/pyo3)
