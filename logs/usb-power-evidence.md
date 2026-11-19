# USB and battery evidence, November 18-19, 2026

These are confirmed results from the Framework Laptop 13 Pro on Sequoia 15.8.3 24H241, Darwin 24.6.0. Raw archives remain in the private investigation workspace. This public summary omits serial numbers, MAC addresses and unrelated registry data. No Apple kernel collection or reconstructed driver binary is published here.

## Reconnect timing

Time is measured from the driver connection-handler entry to successful HID event-driver startup, not from physical insertion or the first input report.

| Attempt | Prompted trial intervals | Null reset results | Notes |
| --- | --- | --- | --- |
| 106 | 0.248, 8.177, 8.189 s | 0, 6, 6 | Another unprompted cycle took 8.166 s with six null results. |
| 107 | 1.951, 0.502, 0.546 s | 1, 0, 0 | Minimum 350 ms window scoped to HS02 at 14200000. |
| 108 | 0.531, 0.542, 0.517, 0.529, 0.523, 0.540 s | 0 in all six | Same policy, with additional status tracing. The host trace limit was reached before trial 6, but reset/enumeration and HID logs continued. |

Attempt 108 human insertion-to-pointer marks ranged from 0.985 to 1.552 s and include reaction time. Successful captured host creation used status 0x1102. Zero failures in one six-trial run does not invalidate the occasional failure in Attempt 107 or establish general USB stability.

## Battery transitions, Attempt 109

| Phase | pmset | External power | Charging | Voltage |
| --- | --- | --- | --- | --- |
| Connected | 30%, AC Power | Yes | Yes | 15.742 V |
| Unplugged | 32%, Battery Power | No | No | 15.337 V |
| Reconnected | 33%, AC Power | Yes | Yes | 15.964 V |

SMCBatteryManager 1.3.8 found one battery and one adapter. Full capacity was 4514 mAh, design capacity 4640 mAh and cycle count 42. The withdrawn frozen-status report is not recorded as a confirmed defect. Driver source defines quick/normal polling of 1/60 seconds plus notifications. UI latency was not measured. Sleep/wake, CPU power efficiency and measurement accuracy were not established.

## Original capture archive digests

These SHA-256 values identify the private source captures used for the journal and technical report.

| Attempt | Archive | SHA-256 |
| --- | --- | --- |
| 104 | `attempt-104-20261118-151951-823.tgz` | `8e890125020c766d602657cfa82784ae5e5ead1f82fe709938045c0af2b0f099` |
| 105 | `attempt-105-20261118-224601-664.tgz` | `d138cda7f82f66faeaec00480dd80d36e04c40254e89a0d98a49980f35b37792` |
| 106 | `attempt-106-20261119-095529-725.tgz` | `d1b793dc8f2f7aa78571faf90299e6b1db844802e542126282be5658d23e30f7` |
| 107 | `attempt-107-20261119-101146-670.tgz` | `3d94e26e728b602548e3d3d58bdc5d1a39f5ff5dd0095cafdb28dfa416fba5b4` |
| 108 | `attempt-108-20261119-102959-755.tgz` | `619d23ba15a9547e6f1b89568f9aac30b457c2743d9e58c74a1b7766e9f5dd30` |
| 109 | `attempt-109-20261119-104839-669.tgz` | `2cee2e504cc2928524c8c525be96db561f7329c49e7e86f6d1525655a241c479` |

All staged EFIs passed matching OpenCore 1.0.8 validation and file-reference checks. Mounted-file matches were recorded for Attempts 104-105, but were not independently captured for Attempts 106-109. Staging validation is distinct from proof of deployed bytes. Attempts 106-109 include explicit active-mode and successful hook/plugin evidence in the boot captures.

Credits and full context are in the [technical findings](../TECHNICAL-FINDINGS.md#credits-and-references).
