# TESTPLAN: Ryzentosh-WIRELESS

The full per-row plan, pass criteria and baseline are in [ADR-0001](decisions/ADR-0001-ryzentoshwireless-architecture.md#test-plan-per-compatibility-matrix-row).
Baseline: MtkWireless 1.1.3 on the owner's machine (control plane `logs-local/ewifi2/`): 97.9/50.7 Mbps, 308 RPM, Tx 867, 0 % gateway ping loss in 30 min.

| Parity target | Test | Host or device | Status |
|---|---|---|---|
| Wi-Fi throughput/latency ≥ baseline | networkQuality ×3, 30-min ping, 1-h idle | device | planned (W4) |
| AirDrop both ways, also with the AP on a DFS channel | 10 photos + 1 GB each way, ch 52 and ch 36–48 | device | planned (W6); owner test T1 open |
| Handoff / Universal Clipboard ≥ 19/20 | 10 tries each way, link type logged | device | planned (W6, W8); owner test T3 open |
| iPhone Bluetooth pair/connect from either side | pair/unpair/re-pair ×3, HCI trace | device | planned (W9); owner test T2 open |
| BLE advertising never fails | log counters per hour | device | planned (W8) |
| Firmware headers, MCU TLVs, BA reorder, regulatory, AWDL window maths, ABI guard, bluetoothd signatures | unit tests | host | planned (W2-W7) |
