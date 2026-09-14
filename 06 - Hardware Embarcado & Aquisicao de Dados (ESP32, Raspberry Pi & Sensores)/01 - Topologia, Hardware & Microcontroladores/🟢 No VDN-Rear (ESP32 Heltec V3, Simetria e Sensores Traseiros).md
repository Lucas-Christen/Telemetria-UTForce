---
tipo: nota-tecnica
area: faculdade
projeto: "UTForce E-Racing"
status: desenvolvimento
data_atualizacao: 2026-09-14
modulo: "06 - Hardware Embarcado & Aquisição de Dados"
submodulo: "01 - Topologia, Hardware & Microcontroladores"
documento: "Nó VDN-Rear (ESP32-S3-DevKitC-1, simetria e sensores traseiros)"
autor: "Lucas Fernandes Christen"
fonte_canonica: "[[🧭 Plano do Sistema de Telemetria (FSAE Eletrico)]]"
tags:
  - esp32-s3
  - devkitc
  - mcp3208
  - pcnt
  - dinamica-veicular
---

# 🟢 Nó VDN-Rear (ESP32-S3-DevKitC-1, Simetria & Sensores Traseiros)

> Mesmo hardware, mesma PCB e mesmo firmware do [[🟢 No VDN-Front (ESP32 Heltec V3, Entradas Digitais e SPI)|VDN-Front]]. Só muda o jumper de ID e, com ele, a faixa de IDs CAN (`0x220–0x22F`). Placa: **ESP32-S3-DevKitC-1** (o nome do arquivo ainda diz "Heltec V3"). Barramento: **CAN-2**.

---

## 🧭 1. Princípio: uma PCB, dois nós

* **Hardware idêntico** — uma placa sobressalente serve para qualquer um dos dois eixos.
* **Firmware idêntico** — um único binário. No boot, o nó lê **GPIO 21**: aberto (pull-up interno) = Front; em GND = Rear. O valor escolhe a base de IDs (`0x200` ou `0x220`) e o rótulo no heartbeat.
* Sem `#define NODE_ID_REAR` em tempo de compilação: dois binários diferentes é uma fonte de erro de campo a menos.

```cpp
const uint16_t CAN_BASE = (digitalRead(21) == HIGH) ? 0x200 : 0x220;
```

---

## 📌 2. Mapeamento de canais — dianteiro × traseiro

| Entrada | VDN-Front | VDN-Rear | Uso em dinâmica |
| :--- | :--- | :--- | :--- |
| MCP3208 CH0 | `damper_pos_fl` | `damper_pos_rl` | Curso de amortecedor → velocidade de amortecedor, ride height derivado, *roll* e *pitch* de suspensão |
| MCP3208 CH1 | `damper_pos_fr` | `damper_pos_rr` | idem |
| MCP3208 CH2 | `hub_accel_z_fl` | `hub_accel_z_rl` | ADXL377 — *wheel hop*, espectro de massa não suspensa |
| MCP3208 CH3 | `hub_accel_z_fr` | `hub_accel_z_rr` | idem |
| MCP3208 CH4 | `chassis_accel_z_f` | `chassis_accel_z_r` | ADXL335 — *pitch rate* entre eixos, comparação com o BNO085 do INU |
| PCNT 0 (GPIO 4) | `wheel_speed_fl` | `wheel_speed_rl` | Slip ratio (as traseiras são as trativas) |
| PCNT 1 (GPIO 5) | `wheel_speed_fr` | `wheel_speed_rr` | idem |
| I²C0 (8/9) | `tyre_temp_fl_*` | `tyre_temp_rl_*` | MLX90621 esquerdo |
| I²C1 (17/18) | `tyre_temp_fr_*` | `tyre_temp_rr_*` | MLX90621 direito |

Pinagem completa, condicionamento e faixas: ver [[🟢 No VDN-Front (ESP32 Heltec V3, Entradas Digitais e SPI)]] §2–§3. Não repetir aqui — uma fonte só.

---

## 📤 3. Frames emitidos (CAN-2)

| ID | Nome | Taxa | Espelho de |
| :---: | :--- | :---: | :---: |
| `0x220` | `VDNR_Damper` | 200 Hz | `0x200` |
| `0x221` | `VDNR_Wheel` | 100 Hz | `0x201` |
| `0x222` | `VDNR_Tyre` | 10 Hz | `0x202` |
| `0x22F` | `VDNR_Heartbeat` | 1 Hz | `0x20F` |

Layout de bytes idêntico ao dianteiro — o DBC define uma mensagem por ID mas o decodificador é o mesmo.

---

## ⚡ 4. Terminação do CAN-2

O VDN-Rear é a **ponta traseira do CAN-2**; o VDN-Front é a dianteira. Cada um carrega um resistor de 120 Ω dentro do conector final.

* Com o carro desligado, `CAN-H`–`CAN-L` do CAN-2 deve medir **60 Ω**.
* 120 Ω → uma ponta desconectada. ~40 Ω → alguém soldou um terceiro resistor (INU, Térmico, Logger B e Gateway LoRa **não** têm terminação). 0 Ω → curto.
* O CAN-1 tem as próprias duas terminações (CVW300 e VCU) e é medido separadamente — ver [[🌐 Topologia Macro da Rede CAN 500k e Distribuicao dos Nos]] §3.

---

## 🔍 5. Diferença mecânica que o firmware não vê

As rodas traseiras são as **trativas**. O slip ratio calculado no Pi usa `wheel_speed_rl/rr` contra a velocidade de referência (média das dianteiras ou GPS). Isso não muda nada no VDN-Rear — é responsabilidade do Pi ([[⚙️ Calculos de Dinamica em Tempo Real (Slip, AeroBalance e G-Forces)]]).

---

## 🔗 Próxima Leitura
* [[🔵 No PCU (Controle, Powertrain, APS, Freios e Direcao)|➡️ Nó VCU]]
* [[📐 Ride Height (Laser vs Potenciometro Linear) e Calibracao|📐 Ride height derivado do amortecedor]]
* [[🌐 Arquitetura CAN 2.0B (500 kbps), Transceivers SN65HVD230 e Terminacao|🌐 Camada física CAN]]
