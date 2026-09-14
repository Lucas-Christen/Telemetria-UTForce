---
tipo: nota-tecnica
area: faculdade
projeto: "UTForce E-Racing"
status: desenvolvimento
data_atualizacao: 2026-09-14
modulo: "06 - Hardware Embarcado & Aquisição de Dados"
submodulo: "03 - Barramento CAN & Firmware dos Nós"
documento: "Matriz de Mensagens CAN dos dois barramentos, layout de bytes por frame e regras do DBC"
autor: "Lucas Fernandes Christen"
fonte_canonica: "[[🧭 Plano do Sistema de Telemetria (FSAE Eletrico)]]"
tags:
  - can-bus
  - dbc
  - matriz-can
  - payload
  - little-endian
---

# 📊 Matriz de Mensagens CAN & Dicionário de IDs (DBC Veicular)

> Layout byte a byte de cada frame dos **dois barramentos**. IDs, taxas e DLC vêm de [[🧭 Plano do Sistema de Telemetria (FSAE Eletrico)]] §7; nomes de sinal do [[📊 Dicionario Completo de Canais, Fontes e Taxas de Amostragem (Hz)|Dicionário Canônico]]. Este arquivo é a fonte para gerar `utforce_fsae.dbc`.

> [!warning] O que mudou
> * **Dois barramentos**, dois DBCs (ou um DBC com dois *buses*). A versão anterior tinha um.
> * Faixas de ID redistribuídas: `0x0xx` segurança, `0x1xx` VCU, `0x2xx` VDNs, `0x3xx` inversor, `0x4xx` Orion, `0x5xx` INU, `0x6xx` Térmico, `0x7xx` gateway. `0x6xx` **não é mais** "cálculos do Pi" — o Pi não transmite.
> * **Um frame, uma taxa.** Acabou o `0x200` com roda a 100 Hz e ride height a 10 Hz juntos.
> * Sem `float` IEEE 754 no CAN. Tudo inteiro com fator e offset — decodifica em qualquer ferramenta.
> * IDs terminados em `F` são heartbeats.

---

## 🧭 1. Convenções

| Regra | Valor |
| :--- | :--- |
| Identificador | 11 bits (CAN 2.0A), padrão |
| Byte order | **Little Endian** (Intel) em todos os sinais multi-byte |
| Tipos | `u8 i8 u16 i16 u32 i32`. Físico = raw × fator + offset |
| Valor inválido | Máximo do tipo (`0xFFFF` para u16, `0x7FFF` para i16) = **NaN**. O decodificador trata como ausência |
| Taxa "evento + 1 Hz" | Emitido a 1 Hz **e** imediatamente a cada mudança de bit |
| Ocupação estimada | bits/frame ≈ 55 + 10·DLC (pior caso com *stuffing*) |

---

## 🔴 2. CAN-1 — Trativo (≈ 67 kbps, 13 %)

### 2.1 Segurança

**`0x010 VCU_Safety`** — VCU — evento + 1 Hz — DLC 2

| Byte | Bit | Sinal | Nota |
| :---: | :---: | :--- | :--- |
| 0 | 0 | `sdc_ok` | 1 = circuito de shutdown fechado |
| 0 | 1 | `imd_ok` | |
| 0 | 2 | `ams_ok` | |
| 0 | 3 | `bspd_ok` | |
| 0 | 4 | `air_pos_closed` | |
| 0 | 5 | `air_neg_closed` | |
| 0 | 6 | `precharge_done` | |
| 0 | 7 | `tsal_active` | |
| 1 | 0 | `rtd_active` | |
| 1 | 1–7 | `safety_event_counter` | u7, incrementa a cada emissão por evento |

### 2.2 VCU

**`0x100 VCU_Torque`** — 100 Hz — DLC 6

| Bytes | Sinal | Tipo | Fator | Unidade |
| :---: | :--- | :---: | :---: | :---: |
| 0–1 | `torque_cmd` | i16 | 0,1 | Nm |
| 2–3 | `torque_limit` | i16 | 0,1 | Nm |
| 4 | `drive_mode` | u8 | enum | 0 off · 1 RTD · 2 regen · 3 limp |
| 5 | `inverter_enable` | u8 | bool | |

**`0x101 VCU_APPS`** — 100 Hz — DLC 7

| Bytes | Sinal | Tipo | Fator | Unidade |
| :---: | :--- | :---: | :---: | :---: |
| 0–1 | `apps1_raw` | u16 | 1 | contagem (ADS131M08, 16 MSB) |
| 2–3 | `apps2_raw` | u16 | 1 | contagem (ADS1115) |
| 4–5 | `apps_pct` | u16 | 0,01 | % |
| 6 | flags | u8 | bit0 `apps_implausible` · bit1 `bse_apps_implausible` · bit2 `apps1_fault` · bit3 `apps2_fault` | |

**`0x102 VCU_Brake`** — 100 Hz — DLC 7

| Bytes | Sinal | Tipo | Fator | Unidade |
| :---: | :--- | :---: | :---: | :---: |
| 0–1 | `bse_press_front` | u16 | 0,1 | bar |
| 2–3 | `bse_press_rear` | u16 | 0,1 | bar |
| 4–5 | `brake_pedal_pos` | u16 | 0,1 | mm |
| 6 | flags | u8 | bit0 `brake_active` · bit1 `bse_f_fault` · bit2 `bse_r_fault` · bit3 `pedal_fault` | |

**`0x103 VCU_Steer`** — 100 Hz — DLC 6

| Bytes | Sinal | Tipo | Fator | Unidade |
| :---: | :--- | :---: | :---: | :---: |
| 0–1 | `steer_angle` | i16 | 0,1 | ° (esquerda +) |
| 2–3 | `steer_torque` | i16 | 0,01 | Nm |
| 4–5 | `steer_rate` | i16 | 0,1 | °/s |

**`0x110 VCU_Inputs`** — 10 Hz — DLC 2: byte 0 `regen_level` u8 %; byte 1 `buttons` bitmask (bit0 RTD, bit1 pit-limiter, bit2–7 reserva).

**`0x10F VCU_Heartbeat`** — 1 Hz — DLC 8 — layout comum (§4).

### 2.3 Inversor CVW300 *(pendente: mapear `%SW` no WLP)*

**`0x300 INV_Status`** — 100 Hz — DLC 8

| Bytes | Sinal | Tipo | Fator | Unidade |
| :---: | :--- | :---: | :---: | :---: |
| 0–1 | `motor_speed_rpm` | i16 | 1 | rpm |
| 2–3 | `motor_torque_actual` | i16 | 0,1 | Nm |
| 4–5 | `dc_bus_voltage` | u16 | 0,1 | V |
| 6–7 | `dc_bus_current` | i16 | 0,1 | A |

**`0x301 INV_Thermal`** — 10 Hz + evento — DLC 8

| Bytes | Sinal | Tipo | Fator | Unidade |
| :---: | :--- | :---: | :---: | :---: |
| 0–1 | `motor_temp` | i16 | 0,1 | °C |
| 2–3 | `inverter_igbt_temp` | i16 | 0,1 | °C |
| 4–5 | `inverter_status_word` | u16 | bitmask | conforme WLP |
| 6–7 | `inverter_fault_word` | u16 | bitmask | conforme WLP |

O CVW300 usa o protocolo "CAN Automotivo" da WEG, configurado no WLP: período mínimo 10 ms (→ 100 Hz é o teto), até 4 WORDs por telegrama. Estes layouts são o **alvo** a configurar; não há DBC de fábrica.

### 2.4 Orion BMS — CAN1

**`0x400 BMS_Pack`** — 20 Hz — DLC 8

| Bytes | Sinal | Tipo | Fator | Unidade |
| :---: | :--- | :---: | :---: | :---: |
| 0–1 | `ts_pack_voltage` | u16 | 0,1 | V |
| 2–3 | `ts_pack_current` | i16 | 0,1 | A (descarga +) |
| 4 | `soc_pct` | u8 | 1 | % |
| 5–6 | `dcl` | u16 | 1 | A |
| 7 | `ccl` | u8 | 1 | A |

**`0x401 BMS_Cells`** — 1 Hz — DLC 8

| Bytes | Sinal | Tipo | Fator | Unidade |
| :---: | :--- | :---: | :---: | :---: |
| 0–1 | `cell_v_min` | u16 | 1 | mV |
| 2–3 | `cell_v_max` | u16 | 1 | mV |
| 4 | `cell_v_min_id` | u8 | 1 | 1–24 |
| 5 | `cell_v_max_id` | u8 | 1 | 1–24 |
| 6 | `cell_t_min` | i8 | 1 | °C |
| 7 | `cell_t_max` | i8 | 1 | °C |

**`0x402 BMS_Status`** — evento + 1 Hz — DLC 4: `bms_fault_flags` u16 · `relay_state` u8 (bit0 discharge, bit1 charge, bit2 charger safety) · `charge_mode` u8.

O Orion é totalmente configurável: estes layouts são **o que configurar** na ferramenta da Orion, canal CAN1 ([[🧭 Plano do Sistema de Telemetria (FSAE Eletrico)]] §12).

---

## 🔵 3. CAN-2 — Aquisição (≈ 118 kbps, 24 %)

### 3.1 VDN-Front (`0x200–0x20F`) e VDN-Rear (`0x220–0x22F`, mesmo layout)

**`0x200 VDNF_Damper` / `0x220 VDNR_Damper`** — **200 Hz** — DLC 8

| Bytes | Sinal (Front / Rear) | Tipo | Fator | Unidade |
| :---: | :--- | :---: | :---: | :---: |
| 0–1 | `damper_pos_fl` / `_rl` | u16 | 0,01 | mm |
| 2–3 | `damper_pos_fr` / `_rr` | u16 | 0,01 | mm |
| 4–5 | `hub_accel_z_fl` / `_rl` | i16 | 0,01 | g |
| 6–7 | `hub_accel_z_fr` / `_rr` | i16 | 0,01 | g |

**`0x201 VDNF_Wheel` / `0x221 VDNR_Wheel`** — 100 Hz — DLC 6

| Bytes | Sinal | Tipo | Fator | Unidade |
| :---: | :--- | :---: | :---: | :---: |
| 0–1 | `wheel_speed_fl` / `_rl` | u16 | 0,01 | km/h |
| 2–3 | `wheel_speed_fr` / `_rr` | u16 | 0,01 | km/h |
| 4–5 | `chassis_accel_z_f` / `_r` | i16 | 0,001 | g |

**`0x202 VDNF_Tyre` / `0x222 VDNR_Tyre`** — 10 Hz — DLC 6

| Byte | Sinal | Tipo | Fator | Offset |
| :---: | :--- | :---: | :---: | :---: |
| 0 / 1 / 2 | `tyre_temp_fl_in` / `_mid` / `_out` (Rear: `rl`) | u8 | 1 | −40 °C |
| 3 / 4 / 5 | `tyre_temp_fr_in` / `_mid` / `_out` (Rear: `rr`) | u8 | 1 | −40 °C |

**`0x20F` / `0x22F` Heartbeat** — 1 Hz — DLC 8 — §4.

### 3.2 INU (`0x500–0x50F`)

**`0x500 INU_Accel`** — 100 Hz — DLC 8: `accel_x` i16 ×0,001 g · `accel_y` i16 · `accel_z` i16 · `gyro_yaw_rate` i16 ×0,01 °/s.

**`0x501 INU_Attitude`** — 100 Hz — DLC 8: `gyro_roll_rate` i16 ×0,01 °/s · `gyro_pitch_rate` i16 · `att_roll_angle` i16 ×0,01 ° · `att_pitch_angle` i16 ×0,01 °.

**`0x502 INU_GPS_Pos`** — 10 Hz — DLC 8: `gps_lat` i32 ×1e-7 ° · `gps_lon` i32 ×1e-7 °.

**`0x503 INU_GPS_Vel`** — 10 Hz — DLC 8: `gps_speed` u16 ×0,01 km/h · `gps_heading` u16 ×0,01 ° · `gps_alt` i16 m · `gps_sats` u8 · `gps_fix` u8 (0 none · 2 2D · 3 3D · 4 3D+DGNSS).

**`0x504 INU_TimeSync`** — **1 Hz, emitido na borda do PPS** — DLC 8: `epoch` u32 s (UTC) · `ms` u16 (atraso entre PPS e emissão) · `pps_count` u16.

**`0x50F INU_Heartbeat`** — 1 Hz — DLC 8.

### 3.3 Térmico (`0x600–0x60F`)

**`0x600 THM_Coolant`** — 5 Hz — DLC 8: `coolant_temp_motor_in` · `_motor_out` · `_inv_in` · `_inv_out`, todos i16 ×0,1 °C.

**`0x601 THM_Flow`** — 5 Hz — DLC 7: `coolant_flow_lpm` u16 ×0,1 L/min · `pump_duty_pct` u8 · `fan_duty_pct` u8 · `gearbox_temp` i16 ×0,1 °C · `lv_battery_voltage` u8 ×0,1 V.

**`0x60F THM_Heartbeat`** — 1 Hz — DLC 8.

### 3.4 Orion BMS — CAN2 (resumo)

**`0x410 BMS_Summary`** — 10 Hz — DLC 6: `soc_pct` u8 · `ts_pack_voltage` u16 ×0,1 V · `ts_pack_current` i16 ×0,1 A · `cell_t_max` i8 °C.

### 3.5 Gateway VCU → CAN-2 (`0x700–0x70F`)

**`0x700 GW_Pedals`** — 20 Hz — DLC 8: `apps_pct` u16 ×0,01 % · `bse_press_front` u16 ×0,1 bar · `bse_press_rear` u16 ×0,1 bar · `steer_angle` i16 ×0,1 °. Subamostrado de `0x101/0x102/0x103`.

**`0x701 GW_Safety`** — evento + 1 Hz — DLC 2: cópia byte a byte de `0x010`.

**`0x702 GW_Inverter`** — 20 Hz — DLC 8: cópia de `0x300` (a cada 5º frame).

O gateway é **unidirecional**: nada do CAN-2 é copiado para o CAN-1.

---

## 💓 4. Heartbeat — layout comum (`0x?0F`, 1 Hz, DLC 8)

| Bytes | Sinal | Tipo | Fator | Nota |
| :---: | :--- | :---: | :---: | :--- |
| 0–1 | `uptime_s` | u16 | 2 s | Zera no reset → detecta watchdog |
| 2 | `can_tx_err` | u8 | 1 | TEC do controlador (saturado em 255) |
| 3 | `can_rx_err` | u8 | 1 | REC |
| 4 | `bus_off_count` | u8 | 1 | Desde o boot |
| 5 | `adc_fault_flags` | u8 | bitmask | Um bit por canal fora de faixa (definido por nó) |
| 6 | bits 0–3 `sd_status` · bits 4–7 `fw_version` | u4 + u4 | enum | sd: 0 sem SD · 1 ok · 2 cheio · 3 erro. fw: 0–15 |
| 7 | `free_heap_kb` | u8 | 4 kB | 0–1020 kB |

**Ausência de heartbeat por > 2 s** = nó em falha → display do piloto (VCU e Pi detectam). Nós só-escuta (Logger B, Gateway LoRa) **não emitem** heartbeat no CAN — o Logger B registra o próprio estado no SD; o Gateway LoRa reporta no pacote de rádio.

---

## 🧾 5. Ocupação

| Barramento | Frames/s | Bits/s (pior caso) | Ocupação |
| :--- | :---: | :---: | :---: |
| CAN-1 | ≈ 550 | ≈ 67 000 | **13 %** |
| CAN-2 | ≈ 900 | ≈ 118 000 | **24 %** |

Abaixo de 30 % — sobra para expansão e para a arbitragem não atrasar os frames de 100 Hz do VCU. Se algum dia o CAN-2 passar de 50 %, o primeiro candidato a cair para 100 Hz é `hub_accel_z_*`.

---

## 📄 6. Gerando o DBC

Um arquivo por barramento (`utforce_can1.dbc`, `utforce_can2.dbc`), gerados por script a partir desta nota (ou de um CSV derivado dela) com `cantools`. Convenções no DBC: `BO_` com o nome da coluna "Nome", `SG_` com o nome canônico, `@1+` (little endian, unsigned) ou `@1-`, fator e offset como acima, `VAL_` para enums (`drive_mode`, `gps_fix`, `sd_status`). O mesmo DBC decodifica o log do Pi (BLF), o do Logger B e o do VCU (após conversão para candump).

---

## 🔗 Próxima Leitura
* [[📊 Dicionario Completo de Canais, Fontes e Taxas de Amostragem (Hz)|📊 Dicionário Canônico]]
* [[💻 Arquitetura de Firmware PlatformIO (C++, FreeRTOS e Core Pinning)|💻 Como os nós montam e enviam os frames]]
* [[🔴 Gateway e Logger Raspberry Pi (SocketCAN, Logs BLF e CSV)|🔴 Decodificação no Pi]]
* [[📡 Modulo LoRa SX1262, Bit-Packing Compacto e CRC16|📡 Do CAN-2 ao pacote de rádio]]
