---
tipo: nota-tecnica
area: faculdade
projeto: "UTForce E-Racing"
status: desenvolvimento
data_atualizacao: 2026-09-14
modulo: "06 - Hardware Embarcado & Aquisição de Dados"
submodulo: "02 - Sensores & Condicionamento de Sinais"
documento: "Unidade Inercial BNO085 e GPS NEO-M8N: UBX a 10 Hz, PPS como base de tempo, calibração"
autor: "Lucas Fernandes Christen"
fonte_canonica: "[[🧭 Plano do Sistema de Telemetria (FSAE Eletrico)]]"
tags:
  - imu
  - gps
  - bno085
  - neo-m8n
  - ubx
  - pps
  - sincronismo
---

# 🛰️ Unidade Inercial BNO085 & GPS NEO-M8N com Sincronismo PPS

> Protocolos, configuração e sincronismo do par IMU + GNSS do **INU**. Pinagem e frames estão em [[🟣 No INU (ESP32 T-Beam, GPS NEO-M8N e IMU BNO085 9-DoF)]]; aqui fica o **como configurar** e o **por que** de cada escolha.

> [!warning] O que mudou
> * IMU a **100 Hz** para tudo (era 20/50 Hz misturado). Aceleração **raw** (com gravidade), não a "linear".
> * GPS a 10 Hz exige **uma constelação** (GPS-only) no NEO-M8N — a versão anterior prometia multi-constelação a 10 Hz, o que o módulo não faz.
> * PPS em **GPIO 37** (confirmar na revisão da placa), não 34.
> * `time_sync` (`0x504`) é a **base de tempo de todo o sistema** — a versão anterior tratava como detalhe.

---

## 🛰️ 1. NEO-M8N — configuração UBX

Sequência de mensagens de configuração enviadas pelo INU no boot (UART, 9600 → 115200 baud):

| Passo | Mensagem UBX | Conteúdo | Motivo |
| :---: | :--- | :--- | :--- |
| 1 | `CFG-PRT` | UART1 115200, protocolo de saída **UBX apenas** | Desliga NMEA (texto ASCII, lento de parsear) |
| 2 | `CFG-GNSS` | **GPS + SBAS habilitados; GLONASS, Galileo, BeiDou desligados** | Só assim o M8N sustenta 10 Hz. Com duas constelações cai para 5 Hz |
| 3 | `CFG-RATE` | `measRate` = 100 ms, `navRate` = 1 | 10 soluções/s |
| 4 | `CFG-MSG` | `NAV-PVT` a cada solução | Um pacote com tudo |
| 5 | `CFG-NAV5` | Modelo dinâmico **automotive**, fix 3D apenas | Filtro interno adequado a acelerações de carro |
| 6 | `CFG-TP5` | TIMEPULSE 1 Hz, largura 100 ms, alinhado ao topo do segundo UTC, só com *fix* válido... **ou** sempre ligado | Ver §3 — recomendação: sempre ligado, com flag de validade no frame |
| 7 | `CFG-CFG` | Salvar em BBR/flash | Não reconfigurar a cada boot frio |

**UBX-NAV-PVT** (92 bytes) traz: `iTOW`, ano/mês/dia/h/min/s, `valid`, `tAcc`, `nano`, `fixType`, `flags`, `numSV`, `lon`, `lat` (×1e-7 °), `height`, `hMSL`, `hAcc`, `vAcc`, `velN/E/D`, `gSpeed` (mm/s), `headMot` (×1e-5 °), `sAcc`, `headAcc`, `pDOP`. O INU usa `lat`, `lon`, `gSpeed`, `headMot`, `hMSL`, `numSV`, `fixType` e a hora.

**Antena:** patch ativa 25×25 mm no topo do *roll hoop*, plano de terra ≥ 50 mm, cabo RG174 ≤ 1 m. Longe da antena LoRa (≥ 30 cm) — 915 MHz não interfere em L1, mas o amplificador da antena ativa satura com potência próxima.

---

## 📐 2. BNO085 — relatórios SH-2

O BNO085 tem coprocessador de fusão (SH-2). O INU habilita três relatórios a **100 Hz** (intervalo 10 000 µs):

| Relatório | Canais | Observação |
| :--- | :--- | :--- |
| `SH2_ACCELEROMETER` | `accel_x/y/z` (i16 ×0,001 g) | **Raw, com gravidade.** O Pi subtrai a gravidade usando `att_roll/pitch`. Assim o dado bruto fica no log e é comparável ao ADXL335 do chassi |
| `SH2_GYROSCOPE_CALIBRATED` | `gyro_roll/pitch/yaw_rate` (i16 ×0,01 °/s) | Bias compensado pelo SH-2 |
| `SH2_ROTATION_VECTOR` → quaternion → Euler | `att_roll/pitch_angle` (i16 ×0,01 °) | **Só roll e pitch.** O *yaw* do rotation vector depende do magnetômetro, inútil perto do inversor. Se precisar de *heading*, é o `headMot` do GPS |

**Não habilitar:** `LINEAR_ACCELERATION`, `GAME_ROTATION_VECTOR` com magnetômetro, `STABILITY_CLASSIFIER` — gastam banda I²C sem uso.

**Interface:** I²C 400 kHz, endereço **0x4A** (ou 0x4B com o pino de endereço), INT em **GPIO 13** para ler só quando há dado (evita *polling* e travamento do bus compartilhado com o AXP192).

**Montagem:** eixos ISO 8855 (X frente, Y esquerda, Z cima), o mais perto do CG. Rotação de montagem corrigida por matriz fixa no firmware (`SH2_SET_REORIENTATION` ou multiplicação própria). Base rígida — sem *foam*: o BNO085 já filtra vibração, e espuma introduz ressonância a 20–40 Hz, justamente na banda de interesse.

---

## ⏱️ 3. PPS — a base de tempo do carro

```mermaid
sequenceDiagram
    autonumber
    participant SAT as GPS
    participant NEO as NEO-M8N
    participant ISR as INU · ISR GPIO 37
    participant TSK as INU · task
    participant BUS as CAN-2

    SAT->>NEO: tempo atômico
    NEO->>ISR: borda de subida PPS (topo do segundo)
    Note over ISR: t_pps_ms = millis(); pps_count++
    NEO->>TSK: NAV-PVT do mesmo segundo (~50–80 ms depois)
    Note over TSK: epoch = UTC do PVT; ms = millis() − t_pps_ms
    TSK->>BUS: 0x504 {epoch u32, ms u16, pps_count u16}
```

* **Jitter do PPS:** < 60 ns com *fix*. O que importa aqui é ~1 ms (resolução do `millis()` dos nós), então sobra margem de 4 ordens de grandeza.
* **Cada nó** guarda `(millis_local_no_recebimento, epoch)` do último `0x504` e carimba os frames com `millis_local`. O Pi e o Logger B gravam o instante de chegada. No pós-processamento tudo alinha por `0x504`.
* **Sem *fix*:** o TIMEPULSE do M8N pode ser configurado para pulsar mesmo sem *fix* (`CFG-TP5` flag `lockedOtherSet` = 0 / `isFreq`). Recomendação: **pulsar sempre**, e o campo `fix` do `0x503` diz se o `epoch` é confiável. Assim os nós continuam alinhados entre si na garagem, sem satélite.
* **Sem RTC em nenhum nó.** Não precisa — o GPS é o relógio.

---

## 🧪 4. Calibração

| Sensor | Quando | Como | Onde fica |
| :--- | :--- | :--- | :--- |
| Giroscópio | Todo boot, 5 s parado | Automático no SH-2 (`SH2_CAL_GYRO`) | RAM do BNO085 |
| Acelerômetro | Uma vez, na bancada | 6 posições (±X, ±Y, ±Z), até `calibration status = 3` | `SH2_SAVE_DCD` → flash do BNO085; cópia no NVS |
| Magnetômetro | **Não** | Desligado (`SH2_CAL_MAG` = 0) | — |
| Orientação de montagem | Uma vez, no carro | Carro nivelado: `att_roll` = `att_pitch` = 0 → gravar offset | NVS do INU |
| GPS | — | Nada a calibrar; `hAcc` no PVT diz a qualidade | — |

---

## 🔗 Próxima Leitura
* [[🟣 No INU (ESP32 T-Beam, GPS NEO-M8N e IMU BNO085 9-DoF)|🟣 Nó INU — pinos e frames]]
* [[⚙️ Calculos de Dinamica em Tempo Real (Slip, AeroBalance e G-Forces)|⚙️ Slip angle e G-G a partir do INU]]
* [[📊 Matriz de Mensagens CAN e Dicionario de IDs (DBC Veicular)|📊 0x500–0x504]]
