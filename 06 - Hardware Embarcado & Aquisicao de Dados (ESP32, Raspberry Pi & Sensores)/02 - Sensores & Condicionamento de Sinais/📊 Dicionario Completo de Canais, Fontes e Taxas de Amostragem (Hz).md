---
tipo: nota-tecnica
area: faculdade
projeto: "UTForce E-Racing"
status: desenvolvimento
data_atualizacao: 2026-09-14
modulo: "06 - Hardware Embarcado & Aquisição de Dados"
submodulo: "02 - Sensores & Condicionamento de Sinais"
documento: "Dicionário Canônico de Canais: nome único, tipo, origem, barramento e taxa — mais tabela de aliases dos nomes antigos do Miro"
autor: "Lucas Fernandes Christen"
fonte_canonica: "[[🧭 Plano do Sistema de Telemetria (FSAE Eletrico)]]"
tags:
  - canais
  - dicionario
  - taxas
  - dbc
  - nomenclatura
---

# 📊 Dicionário Canônico de Canais, Fontes & Taxas

> **Um nome por grandeza.** Esta é a lista de canais que existem no sistema — cópia organizada de [[🧭 Plano do Sistema de Telemetria (FSAE Eletrico)]] §4, com a tabela de **aliases** que traduz os nomes do Miro (`X_WHL`, `Hub Pos`, `delta Otake`…) para os canônicos. O software de análise exibe o alias que quiser; o DBC, o log e o firmware usam só o nome canônico.

> [!warning] O que mudou em relação à lista do Miro
> * **Eliminados:** 26 canais de combustão, 10 de P2P/pit-limiter, 3 `Transmission Pressure`, 3 ride height laser, 36 nomes duplicados (detalhe em [[📐 Especificacao do Sistema de Aquisicao (FSAE Eletrico)]] §3 e [[📋 Revisao da Arquitetura Planejada (Miro)]] §5).
> * **Adicionados:** estados do circuito de shutdown, canais do CVW300 e do Orion, arrefecimento, heartbeats, `time_sync`.
> * **Taxas** seguem a física do sinal, não a tabela de 20/10/2/1 Hz do Miro: amortecedor a 200 Hz, tudo de pedal a 100 Hz, GPS a 10 Hz.
> * **Um frame, uma taxa.** Canais com taxas diferentes nunca compartilham frame.

---

## ⏱️ 1. Classes de taxa

| Taxa | Fenômeno | Canais |
| :---: | :--- | :--- |
| **200 Hz** | Suspensão e massa não suspensa (ressonância 12–18 Hz; picos de velocidade de amortecedor de poucos ms) | `damper_pos_*`, `hub_accel_z_*` |
| **100 Hz** | Entradas do piloto, powertrain, rodas, IMU, chassi | APPS, BSE, pedal, direção, torque, `motor_speed_rpm`, `dc_bus_*`, `wheel_speed_*`, `chassis_accel_z_*`, `accel_*`, `gyro_*`, `att_*` |
| **20 Hz** | Acumulador (crítico) e cópias do gateway | `ts_pack_*`, `soc_pct`, `dcl`/`ccl` no CAN-1; `GW_Pedals`, `GW_Inverter` |
| **10 Hz** | GPS, térmicos de inversor/motor, pneu, resumo do Orion, `regen_level` | `gps_*`, `motor_temp`, `inverter_igbt_temp`, `tyre_temp_*`, `BMS_Summary` |
| **5 Hz** | Arrefecimento | `coolant_*`, `pump_duty_pct`, `fan_duty_pct`, `gearbox_temp` |
| **1 Hz + evento** | Estados de segurança, células, heartbeats, tempo | `sdc_ok`…`rtd_active`, `cell_v_*`, `bms_fault_flags`, `*_heartbeat`, `time_sync`, `gps_sats/fix`, `lv_battery_voltage` |

---

## 📋 2. Canais adquiridos

### 2.1 Segurança e trativo — VCU, CAN-1

| Canal | Tipo | Origem | Taxa | Frame |
| :--- | :--- | :--- | :---: | :---: |
| `sdc_ok` | bit | Entrada isolada do circuito de shutdown | evento + 1 Hz | `0x010` |
| `imd_ok` | bit | Saída OK do IMD (via PCF8574) | evento + 1 Hz | `0x010` |
| `ams_ok` | bit | Saída discreta do Orion | evento + 1 Hz | `0x010` |
| `bspd_ok` | bit | Latch do BSPD | evento + 1 Hz | `0x010` |
| `air_pos_closed` / `air_neg_closed` | bit | Contato auxiliar dos AIRs | evento + 1 Hz | `0x010` |
| `precharge_done` | bit | Relé de pré-carga | evento + 1 Hz | `0x010` |
| `tsal_active` | bit | Lógica do TSAL | evento + 1 Hz | `0x010` |
| `rtd_active` | bit | Máquina de estados do VCU | evento + 1 Hz | `0x010` |
| `apps1_raw` / `apps2_raw` | u16 | ADS131M08 AIN0 / ADS1115 A0 | 100 Hz | `0x101` |
| `apps_pct` | u16 ×0,01 % | Após plausibilidade | 100 Hz | `0x101` |
| `apps_implausible` | bit | Δ > 10 % por > 100 ms | 100 Hz | `0x101` |
| `bse_press_front` / `bse_press_rear` | u16 ×0,1 bar | Transdutores 0–100 bar | 100 Hz | `0x102` |
| `brake_pedal_pos` | u16 ×0,1 mm | Pot linear do pedal | 100 Hz | `0x102` |
| `bse_apps_implausible` | bit | APPS > 25 % com freio | 100 Hz | `0x101` |
| `steer_angle` | i16 ×0,1 ° | AS5600 | 100 Hz | `0x103` |
| `steer_torque` | i16 ×0,01 Nm | Célula + INA333 | 100 Hz | `0x103` |
| `steer_rate` | i16 ×0,1 °/s | Derivado no VCU | 100 Hz | `0x103` |
| `torque_cmd` / `torque_limit` | i16 ×0,1 Nm | VCU | 100 Hz | `0x100` |
| `regen_level` | u8 % | Pot de painel (ADS1115 A1) | 10 Hz | `0x110` |

### 2.2 Powertrain — CVW300, CAN-1 *(pendente: mapear `%SW` no WLP)*

| Canal | Tipo | Taxa | Frame |
| :--- | :--- | :---: | :---: |
| `motor_speed_rpm` | i16 | 100 Hz | `0x300` |
| `motor_torque_actual` | i16 ×0,1 Nm | 100 Hz | `0x300` |
| `dc_bus_voltage` | u16 ×0,1 V | 100 Hz | `0x300` |
| `dc_bus_current` | i16 ×0,1 A | 100 Hz | `0x300` |
| `motor_temp` / `inverter_igbt_temp` | i16 ×0,1 °C | 10 Hz | `0x301` |
| `inverter_status_word` / `inverter_fault_word` | u16 | 10 Hz + evento | `0x301` |

### 2.3 Acumulador — Orion, CAN-1 (crítico) e CAN-2 (resumo)

| Canal | Tipo | Barramento | Taxa | Frame |
| :--- | :--- | :---: | :---: | :---: |
| `ts_pack_voltage` | u16 ×0,1 V | CAN-1 / CAN-2 | 20 / 10 Hz | `0x400` / `0x410` |
| `ts_pack_current` | i16 ×0,1 A | CAN-1 / CAN-2 | 20 / 10 Hz | `0x400` / `0x410` |
| `soc_pct` | u8 % | CAN-1 / CAN-2 | 20 / 10 Hz | `0x400` / `0x410` |
| `dcl` / `ccl` | u16 A / u8 A | CAN-1 | 20 Hz | `0x400` |
| `cell_v_min` / `cell_v_max` | u16 mV | CAN-1 | 1 Hz | `0x401` |
| `cell_v_min_id` / `cell_v_max_id` | u8 | CAN-1 | 1 Hz | `0x401` |
| `cell_t_min` / `cell_t_max` | i8 °C | CAN-1 (CAN-2 só `t_max`) | 1 Hz / 10 Hz | `0x401` / `0x410` |
| `bms_fault_flags` | u16 | CAN-1 | evento + 1 Hz | `0x402` |

### 2.4 Dinâmica — VDN-Front / VDN-Rear, CAN-2

| Canal | Tipo | Sensor | Taxa | Frame |
| :--- | :--- | :--- | :---: | :---: |
| `damper_pos_[fl,fr,rl,rr]` | u16 ×0,01 mm | Pot linear 75 mm | **200 Hz** | `0x200` / `0x220` |
| `hub_accel_z_[fl,fr,rl,rr]` | i16 ×0,01 g | ADXL377 (±200 g) | 200 Hz | `0x200` / `0x220` |
| `wheel_speed_[fl,fr,rl,rr]` | u16 ×0,01 km/h | TLE4922 + PCNT | 100 Hz | `0x201` / `0x221` |
| `chassis_accel_z_[f,r]` | i16 ×0,001 g | ADXL335 (±3 g) | 100 Hz | `0x201` / `0x221` |
| `tyre_temp_[fl,fr,rl,rr]_[in,mid,out]` | u8 + offset −40 °C | MLX90621 | 10 Hz | `0x202` / `0x222` |

### 2.5 Inercial e posição — INU, CAN-2

| Canal | Tipo | Taxa | Frame |
| :--- | :--- | :---: | :---: |
| `accel_[x,y,z]` | i16 ×0,001 g (raw) | 100 Hz | `0x500` |
| `gyro_yaw_rate` | i16 ×0,01 °/s | 100 Hz | `0x500` |
| `gyro_roll_rate` / `gyro_pitch_rate` | i16 ×0,01 °/s | 100 Hz | `0x501` |
| `att_roll_angle` / `att_pitch_angle` | i16 ×0,01 ° | 100 Hz | `0x501` |
| `gps_lat` / `gps_lon` | i32 ×1e-7 ° | 10 Hz | `0x502` |
| `gps_speed` / `gps_heading` / `gps_alt` | u16 ×0,01 km/h / u16 ×0,01 ° / i16 m | 10 Hz | `0x503` |
| `gps_sats` / `gps_fix` | u8 | 10 Hz (no mesmo frame) | `0x503` |
| `time_sync` (`epoch`, `ms`, `pps_count`) | u32 + u16 + u16 | **1 Hz, borda do PPS** | `0x504` |

### 2.6 Arrefecimento — nó Térmico, CAN-2

| Canal | Sensor | Taxa | Frame |
| :--- | :--- | :---: | :---: |
| `coolant_temp_motor_[in,out]` / `coolant_temp_inv_[in,out]` | NTC 10 k ou PT1000 | 5 Hz | `0x600` |
| `coolant_flow_lpm` | Vazão Hall (PCNT) | 5 Hz | `0x601` |
| `pump_duty_pct` / `fan_duty_pct` | Leitura do PWM | 5 Hz | `0x601` |
| `gearbox_temp` | NTC 10 k no redutor | 5 Hz | `0x601` |
| `lv_battery_voltage` | Divisor → ADS1115 | 5 Hz (no mesmo frame) | `0x601` |

### 2.7 Diagnóstico — todos os nós, `0x?0F`, 1 Hz

`uptime_s`, `can_tx_err`, `can_rx_err`, `bus_off_count`, `adc_fault_flags`, `sd_status`, `fw_version`, `free_heap_kb`.

---

## 🧮 3. Canais derivados (Pi, não adquiridos)

| Canal | Fórmula / fonte | Taxa |
| :--- | :--- | :---: |
| `damper_vel_*` | d(`damper_pos`)/dt | 200 Hz |
| `ride_height_[f,r]`, `rake`, `roll_angle_susp`, `pitch_angle_susp` | `damper_pos` × *motion ratio* ([[📐 Ride Height (Laser vs Potenciometro Linear) e Calibracao]]) | 200 Hz |
| `vehicle_speed` | Média das rodas não trativas (FL, FR), GPS como referência | 100 Hz |
| `slip_ratio_*` | (`wheel_speed_i` − `vehicle_speed`) / max(…) | 100 Hz |
| `slip_angle_[f,r]` | Modelo bicicleta com `steer_angle`, `gyro_yaw_rate`, `vehicle_speed` | 100 Hz |
| `accel_lin_[x,y,z]` | `accel_*` − gravidade projetada por `att_*` | 100 Hz |
| `brake_bias_front` | `bse_front` / (`bse_front` + `bse_rear`) | 100 Hz |
| `energy_consumed_kwh` | ∫ `ts_pack_voltage` × `ts_pack_current` dt | 1 Hz |
| `lap_distance`, `lap_time_delta`, `lap_number` | GPS + linha de chegada | 10 Hz |

---

## 🔁 4. Aliases — nome do Miro → canônico

Para o software de análise exibir o nome que a equipe conhece. **Só exibição.**

| Miro / notas antigas | Canônico |
| :--- | :--- |
| `Throttle Pos`, `TPS`, `APS` | `apps_pct` |
| `Brk_Press_Fnt` / `Brk_Press_Rear`, `Brake Press F/R` | `bse_press_front` / `bse_press_rear` |
| `Brake Pos` | `brake_pedal_pos` |
| `Steered Angle`, `Steer` | `steer_angle` |
| `Steer Torq`, `Steer Torque` | `steer_torque` |
| `Pad_Reg_val`, `Pad_Reg` | `regen_level` |
| `Wheel Speed FL…`, `X_WHL_*`, `WS_*` | `wheel_speed_*` |
| `Damper Pos *`, `Susp Position`, `Hub Pos`, `SUSP_*` | `damper_pos_*` |
| `Ride Height Front/Rear`, `RH_*`, `X_CHZ_*_LSR` | `ride_height_[f,r]` (**derivado**) |
| `FL hub az`, `Hub-az *`, `hub az lateral` | `hub_accel_z_*` (é **vertical**, o rótulo "lateral" do Miro está errado) |
| `Ft acc_z` / `Rear acc_z` | `chassis_accel_z_[f,r]` |
| `Tyre Temp *` | `tyre_temp_*_mid` (média das 3 zonas para compatibilidade) |
| `G Force Lat` / `Long` / `Vertical`, `Gx/Gy/Gz` | `accel_[y,x,z]` (ordem ISO: x = longitudinal) |
| `Yaw Rate`, `Roll rate`, `Pitch rate` | `gyro_[yaw,roll,pitch]_rate` |
| `Roll angle`, `Pitch angle` | `att_[roll,pitch]_angle` |
| `Speed`, `speed`, `vehicle_speed_x100` | `vehicle_speed` (derivado) ou `gps_speed` |
| `Lap Distance`, `X/Y/Z Circ Pos`, `TimeCorr`, `DistCorr`, `Lap Time delta Otake` | `lap_distance`, `gps_lat/lon`, `lap_time_delta` (derivados) |
| `Damper Vel *` | `damper_vel_*` (derivado) |
| `X_Rake_Calc` | `rake` (derivado) |
| `Transmission Temp` | `gearbox_temp` |
| `Transmission Press x/y/z` | **eliminado** |
| `battery_voltage`, `HV Volt` | `ts_pack_voltage` |
| `battery_current` | `ts_pack_current` |
| `battery_soc` | `soc_pct` |
| `battery_max_temp` | `cell_t_max` |
| `F_Aero_*`, `Aerobalance`, `ClA`, `CdA` | **Removidos do sistema embarcado** — são modelo de pós-processamento com dados de túnel/CFD, não telemetria |

---

## 🔗 Próxima Leitura
* [[📊 Matriz de Mensagens CAN e Dicionario de IDs (DBC Veicular)|📊 Layout de bytes de cada frame]]
* [[⚙️ Calculos de Dinamica em Tempo Real (Slip, AeroBalance e G-Forces)|⚙️ Como os derivados são calculados]]
* [[📡 Modulo LoRa SX1262, Bit-Packing Compacto e CRC16|📡 Quais canais vão pelo rádio]]
