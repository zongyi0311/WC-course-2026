# Lab 6：NTN Initial Access 與 RRC Connection Setup（學生版）

## 1. 實驗目標

本實驗使用 OAI RFsimulator 建立透明式 LEO 衛星通道，觀察 UE 如何在具有長傳播延遲與 Doppler shift 的 NTN 環境中完成同步、讀取系統資訊、執行 Random Access，並建立 RRC connection。

本實驗觀察到 `RRCSetupComplete` 與 `RRC_CONNECTED` 為止，不分析後續 NAS Registration、PDU Session 或使用者資料流量。

核心流程如下：

```text
載入 LEO satellite channel model
→ Cell search / initial synchronization
→ MIB / SIB1 / SIB19
→ Msg1 PRACH
→ Msg2 RAR
→ Msg3 RRCSetupRequest
→ Msg4 RRCSetup
→ RRCSetupComplete
→ RRC_CONNECTED
```

## 2. 本實驗要產生的資料

本實驗要求學生自行啟動 gNB、T-tracer recorder 與 NR UE，並產生同一次執行的下列檔案：

| 檔案 | 用途 |
|---|---|
| `Lab6-NTN-ue.log` | 觀察 initial synchronization、SIB19、satellite delay、Doppler、Random Access 與 RRC state。 |
| `Lab6-NTN-gnb.log` | 必要時交叉確認 gNB、LEO channel model 與 RFsimulator 狀態。 |
| `Lab6-NTN-macnr.pcap` | 使用 Wireshark 觀察 MIB、SIB1、SIB19、RAR 與 RRC messages。 |

學生必須先完成資料收集，確認 RRC Setup 成功，再使用自己產生的 log 與 PCAP 完成後續分析。

## 3. 執行前提

開始實驗前，需先確認：

1. OAI gNB 與 NR UE 已完成編譯。
2. OAI 5GC 已啟動，AMF 位址為 `192.168.70.133`。
3. 主機在 Core network 使用的位址為 `192.168.70.129`。
4. 下列指令皆從 `openairinterface5g` 根目錄開始執行。

本實驗的 NTN/RRC 分析不延伸到 Core procedure；啟動 Core 是為了讓一般 SA UE 流程能穩定執行。

## 4. 啟動 NTN 系統並收集資料

### 4.1 建立輸出目錄

```bash
cd openairinterface5g
mkdir -p lab6-artifacts
```

執行時需要三個終端，啟動順序為：

```text
Terminal A：gNB
Terminal B：T-tracer recorder
Terminal C：NR UE
```

### 4.2 Terminal A：啟動 NTN gNB

在 `openairinterface5g` 根目錄執行：

```bash
sudo -E env RFSIMULATOR=server stdbuf -oL -eL \
  ./cmake_targets/ran_build/build/nr-softmodem \
  -O ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-leo.conf \
  --rfsim \
  --gNBs.[0].plmn_list.[0].mcc 208 \
  --gNBs.[0].plmn_list.[0].mnc 95 \
  --gNBs.[0].tracking_area_code 40960 \
  --gNBs.[0].amf_ip_address.[0].ipv4 192.168.70.133 \
  --gNBs.[0].NETWORK_INTERFACES.GNB_IPV4_ADDRESS_FOR_NG_AMF 192.168.70.129 \
  --gNBs.[0].NETWORK_INTERFACES.GNB_IPV4_ADDRESS_FOR_NGU 192.168.70.129 \
  --T_stdout 2 \
  2>&1 | tee lab6-artifacts/Lab6-NTN-gnb.log
```

看到下列訊息後，保留此終端並進入下一步：

```text
T tracer: waiting for connection on 0.0.0.0:2021
```

### 4.3 Terminal B：啟動 T-tracer recorder

開啟另一個終端，進入 `openairinterface5g` 根目錄後執行：

```bash
./cmake_targets/ran_build/build/common/utils/T/tracer/record \
  -d common/utils/T/T_messages.txt \
  -o lab6-artifacts/Lab6-NTN.raw \
  -on WIRESHARK
```

recorder 連線後，gNB 將繼續啟動。等 Terminal A 顯示下列訊息：

```text
Running as server waiting opposite rfsimulators to connect
```

### 4.4 Terminal C：啟動 NTN NR UE

在第三個終端進入 `openairinterface5g` 根目錄後執行：

```bash
sudo -E stdbuf -oL -eL \
  ./cmake_targets/ran_build/build/nr-uesoftmodem \
  -O targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf \
  --band 254 \
  -C 2488400000 \
  --CO -873500000 \
  -r 25 \
  --numerology 0 \
  --ssb 60 \
  --rfsim \
  --rfsimulator.[0].serveraddr 127.0.0.1 \
  --rfsimulator.[0].prop_delay 20 \
  --rfsimulator.[0].options chanmod \
  --time-sync-I 0.1 \
  --ntn-initial-time-drift -46 \
  --initial-fo 57340 \
  --cont-fo-comp 2 \
  2>&1 | tee lab6-artifacts/Lab6-NTN-ue.log
```

`prop_delay` 與 `chanmod` 不可省略；UE 必須載入與 gNB 相符的 LEO channel model，才能在模擬的 delay 與 Doppler 條件下完成 PBCH decoding。

### 4.5 確認 RRC Setup 結果

在第四個終端或 UE 停止後執行：

```bash
grep -nE \
'SAT_LEO|propagation delay|Doppler|Initial sync successful|SIB1 decoded|SIB19|4-Step RA procedure succeeded|Received NR_RRCSetup|State = NR_RRC_CONNECTED|Generating RRCSetupComplete' \
lab6-artifacts/Lab6-NTN-ue.log
```

本實驗至少需找到以下成功證據：

```text
Initial sync successful
Found SIB19
4-Step RA procedure succeeded
Received NR_RRCSetup
State = NR_RRC_CONNECTED
Generating RRCSetupComplete
```

### 4.6 停止順序

完成後，在各自終端依序按 `Ctrl+C`：

1. NR UE
2. T-tracer recorder
3. gNB

### 4.7 將 T-tracer raw 轉成 PCAP

回到 `openairinterface5g` 根目錄執行：

```bash
./cmake_targets/ran_build/build/common/utils/T/tracer/macpdu2wireshark \
  -d common/utils/T/T_messages.txt \
  -i lab6-artifacts/Lab6-NTN.raw \
  -no-bind \
  -to-file lab6-artifacts/Lab6-NTN-macnr.pcap
```

確認檔案：

```bash
ls -lh lab6-artifacts/Lab6-NTN*
```

應至少產生：

```text
Lab6-NTN-gnb.log
Lab6-NTN-ue.log
Lab6-NTN.raw
Lab6-NTN-macnr.pcap
```

## 5. 繳交方式

每個步驟都必須使用自己產生的 log 或 PCAP。截圖需包含足以辨識證據的 log 行、Wireshark frame 或展開欄位；截圖下方附上簡單解釋，不需要在圖片上畫框或箭頭。

## 6. Task 1：LEO Channel 與 Initial Synchronization — 15 分

這個 Task 使用 UE log，確認 UE 確實載入 LEO satellite channel model，並在 delay 與 Doppler 存在時完成 initial synchronization。這些是 RFsimulator 與 UE PHY 的內部執行狀態，不能只靠 MAC-NR PCAP 證明。

### Step 1：找出 LEO channel 與同步證據

在 `openairinterface5g` 根目錄執行：

```bash
grep -nE \
'SAT_LEO_TRANS|propagation delay|Downlink delay|Initial sync successful' \
lab6-artifacts/Lab6-NTN-ue.log | head -n 10
```

截圖必須同時包含：

- `SAT_LEO_TRANS`
- satellite orbit height
- `propagation delay`
- 第一筆 `Downlink delay` 與 `Doppler shift SAT->UE`
- `Initial sync successful` 與 PCI

根據自己的 log 填入：

```text
Satellite orbit height = ______ km
Configured propagation delay = ______ ms
Observed downlink delay = ______ ms
Observed DL Doppler shift = ______ kHz
PCI = ______
```

附上截圖並簡單回答：

1. `propagation delay` 與第一筆 `Downlink delay` 各代表什麼？為什麼兩個數值不一定完全相同？
2. 哪一行證明 UE 已成功找到並同步到 cell？

配分：完整 log 截圖 3 分；五個數值正確 5 分；兩題解釋合理 7 分。

## 7. Task 2：NR 與 NTN System Information — 25 分

### Step 1：找 MIB 與 SIB1 — 10 分

在 Wireshark 開啟自己產生的 `Lab6-NTN-macnr.pcap`，輸入：

```wireshark
nr-rrc.mib_element || nr-rrc.systemInformationBlockType1_element
```

各選擇一筆 MIB 與 SIB1，展開下列欄位：

```text
MIB：
MAC-NR BCCH PDU
→ NR Radio Resource Control (RRC) protocol
→ BCCH-BCH-Message
→ mib

SIB1：
MAC-NR BCCH PDU
→ NR Radio Resource Control (RRC) protocol
→ BCCH-DL-SCH-Message
→ systemInformationBlockType1
```

提供 MIB 與 SIB1 各一張截圖，並填入：

```text
MIB frame number = ______
MIB transport channel = ______
SIB1 frame number = ______
SIB1 transport channel = ______
SIB1 trackingAreaCode = ______
```

在截圖下方簡單回答：

1. 為什麼 PCAP 中會反覆出現 MIB 與 SIB1，而不是只出現一次？
2. 即使本實驗使用 NTN，UE 為什麼仍需要先取得 MIB 與 SIB1，才能繼續接入？

配分：兩張展開正確的截圖 4 分；五個欄位正確 3 分；兩題解釋合理 3 分。

### Step 2：找 SIB19 — 15 分

先在 Wireshark 的 Display Filter 輸入：

```wireshark
nr-rrc
```

按 `Cmd + F`（macOS）或 `Ctrl + F`（Windows/Linux），選擇 `Packet details` 與 `String`，搜尋：

```text
sib19
```

找到後展開：

```text
MAC-NR BCCH PDU
→ NR Radio Resource Control (RRC) protocol
→ BCCH-DL-SCH-Message
→ systemInformation
→ sib-TypeAndInfo
→ sib19-v1700
→ ntn-Config-r17
```

截圖需顯示 SIB19 階層與 `ntn-Config-r17` 的主要欄位。填入：

```text
SIB19 frame number = ______
Transport channel = ______
ntn-UlSyncValidityDuration-r17 = ______
cellSpecificKoffset-r17 = ______
ta-Common-r17 = ______
ta-CommonDrift-r17 = ______
ephemerisInfo-r17 type = ______
```

接著從 UE log 確認 UE 收到 SIB19 並計算 NTN compensation：

```bash
grep -n -m1 'Found SIB19' \
  lab6-artifacts/Lab6-NTN-ue.log

grep -n -m1 'timing_advance_ntn:' \
  lab6-artifacts/Lab6-NTN-ue.log | fold -w 110
```


提供一張 Wireshark 截圖與一張 UE log 截圖，並簡單回答：

1. 相較於 MIB 與 SIB1，SIB19 額外提供哪一類資訊？
2. NTN UE 為什麼需要 timing/drift 與 satellite position/velocity 資訊？
3. Wireshark 的 `cellSpecificKoffset-r17` 與 UE log 的 `k_offset` 是否互相對應？這個比對能證明什麼？

配分：SIB19 Wireshark 截圖 3 分；UE log 截圖 2 分；七個欄位正確 5 分；三題解釋合理 5 分。

## 8. Task 3：Random Access — 20 分

### Step 1：找 Msg1 PRACH — 10 分

先從 UE log 找出 PRACH transmission：

```bash
grep -nE 'PRACH' \
  lab6-artifacts/Lab6-NTN-ue.log
```

UE 可能因為第一次接入未完成而送出多次 PRACH。記錄 log 中每次實際送出的 `frame.slot` 與 `preambleIndex`，再找出最後與 gNB tracer、RAR 相符的那一次：

```text
Number of PRACH transmissions in UE log = ______
Successful-chain PRACH frame.slot = ______
Successful-chain preambleIndex = ______
```

接著在 Wireshark 清除 Display Filter，按 `Cmd + F` 或 `Ctrl + F`，選擇 `Packet details`、`String`，搜尋：

```text
RACH Preamble chosen
```

記錄：

```text
PRACH tracer frame number = ______
RAPID = ______
RACH Attempt Number = ______
```

提供 UE log 與 Wireshark 各一張截圖，並簡單回答：

1. UE log 的哪一個 `preambleIndex` 與 gNB tracer 的 `RAPID` 相同？這個比對代表什麼？
2. 如果 UE log 有多次 PRACH，但 gNB tracer 只出現其中一個 RAPID，能否宣稱其他 PRACH 一定沒有送出？為什麼？

配分：兩張截圖 4 分；六個欄位正確 3 分；兩題解釋合理 3 分。

### Step 2：找 Msg2 RAR — 10 分

在 Wireshark 按 `Cmd + F` 或 `Ctrl + F`，使用 `Packet details`、`String` 搜尋：

```text
MAC-NR RAR
```

若找不到，改搜尋 `RAPID`，並選擇解碼為 `MAC-NR RAR` 的封包。展開 RAR 後填入：

```text
RAR frame number = ______
RA-RNTI = ______
RAPID = ______
Timing Advance = ______
Temporary C-RNTI = ______
```

提供一張展開 RAR 的截圖，並簡單回答：

1. RAR 的 `RAPID` 是否與 Step 1 成功鏈中的 `preambleIndex` 相同？這代表什麼？
2. gNB 在 RAR 中提供 `Temporary C-RNTI` 的用途是什麼？後續應在哪一筆 RRC 訊息中再次看到相同的 RNTI？

配分：RAR 截圖 3 分；五個欄位正確 4 分；兩題解釋合理 3 分。

## 9. 後續分析

接下來的 Task 將使用 UE log 與 Wireshark，依序尋找：

```text
LEO channel / delay / Doppler
→ Initial synchronization
→ MIB / SIB1 / SIB19
→ RRCSetupRequest / RRCSetup / RRCSetupComplete
→ RRC_CONNECTED
```

後續 Task 的操作、繳交項目與配分會隨本次 PCAP 驗證逐步加入。
