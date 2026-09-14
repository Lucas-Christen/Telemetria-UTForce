---
tipo: nota-tecnica
area: faculdade
projeto: "UTForce E-Racing"
status: desenvolvimento
data_atualizacao: 2026-09-14
modulo: "06 - Hardware Embarcado & Aquisição de Dados"
submodulo: "02 - Sensores & Condicionamento de Sinais"
documento: "Circuitos Condicionadores por ADC (MCP3208, ADS131M08, ADS1115), divisores e filtros RC"
autor: "Lucas Fernandes Christen"
fonte_canonica: "[[🧭 Plano do Sistema de Telemetria (FSAE Eletrico)]]"
tags:
  - condicionamento
  - divisor-resistivo
  - filtro-rc
  - mcp3208
  - ads131m08
  - ads1115
  - hardware
---

# ⚡ Circuitos Condicionadores — Divisores, Filtros RC & ADCs

> Dimensionamento do condicionamento **por par sinal → ADC**. A versão anterior falava em "divisor padrão 33k/12k"; isso não existe. Cada ADC tem uma faixa de entrada diferente e o divisor pertence ao destino. Valores verificados contra datasheet em [[🧭 Plano do Sistema de Telemetria (FSAE Eletrico)]] §5.

> [!danger] Apague o "0,733×" do guia do Miro
> Com 33 k em baixo e 12 k em cima o fator é 33/45 = 0,733. Isso leva 4,5 V a 3,30 V e **satura o ADS131M08 (±1,2 V) em 2,75×**. Com 10 V na entrada dá 7,33 V — acima do AVDD. O fator correto para o ADS131M08 é **12/45 = 0,267**, com o **33 k em cima** (no sinal) e o **12 k em baixo** (para AGND).

---

## 🎛️ 1. Faixa de entrada real de cada ADC

| ADC | Resolução | Faixa de entrada | Referência | Interface | Nó |
| :--- | :---: | :--- | :--- | :--- | :--- |
| **MCP3208** | 12 bit | **0 a V_REF = 3,3 V** (single-ended) | Externa = 3,3 V do LDO | SPI 2 MHz | VDN-Front, VDN-Rear |
| **ADS131M08** | 24 bit, 8 canais **diferenciais simultâneos** | **±1,2 V** (ganho 1). Entradas não podem ir abaixo de AGND − 1,3 V nem acima de AVDD | Interna 1,2 V | SPI | VCU |
| **ADS1115** | 16 bit, 4 canais | ±4,096 V no ganho 1, **mas limitada a V_DD**. Alimentado em **5 V lê 0,5–4,5 V direto** | Interna | I²C 0x48 | VCU (APPS2, regen, LV), Térmico (NTCs) |

O erro da versão anterior foi tratar os três como se tivessem entrada de 0–3,3 V.

---

## 🧮 2. Tabela de condicionamento por canal

| Canal | Sinal do sensor | ADC de destino | Condicionamento | Saída no ADC |
| :--- | :--- | :--- | :--- | :---: |
| `apps1_raw` | 0,5–4,5 V | ADS131M08 AIN0 | Divisor **33 k / 12 k** (0,267) | 0,13–1,20 V |
| `apps2_raw` | 4,5–0,5 V (**invertido**) | **ADS1115 A0** (ADC separado) | **Nenhum** — ADS1115 em 5 V | 4,5–0,5 V |
| `bse_press_front` / `_rear` | 0,5–4,5 V | ADS131M08 AIN1 / AIN2 | 33 k / 12 k | 0,13–1,20 V |
| `steer_torque` | ±5 mV (ponte) | ADS131M08 AIN3, **diferencial** contra REF | INA333, R_G = 499 Ω (G ≈ 201), REF = 1,65 V | ±1,0 V em torno de 1,65 V |
| `brake_pedal_pos` | 0–5 V pot | ADS131M08 AIN4 | Divisor **38 k / 12 k** (0,24) | 0–1,20 V |
| `regen_level` | 0–5 V pot | ADS1115 A1 | Nenhum | 0–5 V |
| `lv_battery_voltage` | 10–15 V | ADS1115 A2 (VCU) ou A3 (Térmico) | Divisor 33 k / 12 k (0,267) | 2,7–4,0 V |
| `damper_pos_*` | 0–5 V pot | MCP3208 CH0 / CH1 | Divisor **12 k / 24 k** (0,667) | 0–3,33 V |
| `hub_accel_z_*` | 0–3,3 V (ADXL377 em 3,3 V) | MCP3208 CH2 / CH3 | Só RC 1 k / 100 nF | 0–3,3 V |
| `chassis_accel_z_*` | 0–3,3 V (ADXL335 em 3,3 V) | MCP3208 CH4 | Só RC 1 k / 100 nF | 0–3,3 V |
| NTC 10 k (arrefecimento, redutor) | Divisor com 10 k para 3,3 V | ADS1115 (Térmico) | O próprio divisor | 0–3,3 V |

Resistores: filme metálico, **1 %**, 0,1 W; nas linhas do APPS e BSE, 0,1 % se disponível (o erro de razão do divisor entra direto na plausibilidade).

---

## 📐 3. Contas

### 3.1 Divisor 33 k / 12 k (→ ADS131M08)

$$\alpha = \frac{R_2}{R_1 + R_2} = \frac{12\,\text{k}}{33\,\text{k} + 12\,\text{k}} = \frac{12}{45} = 0{,}2667$$

$4{,}5\,\text{V} \times 0{,}2667 = 1{,}20\,\text{V}$ — exatamente o fundo de escala positivo. Se quiser margem, 39 k / 12 k dá 0,235 → 1,06 V (88 % do FSR).

$$R_{Th} = R_1 \parallel R_2 = \frac{33 \times 12}{45}\,\text{k}\Omega = 8{,}8\,\text{k}\Omega$$

### 3.2 Divisor 12 k / 24 k (→ MCP3208)

$$\alpha = \frac{24}{36} = 0{,}667 \quad\Rightarrow\quad 5{,}0\,\text{V} \to 3{,}33\,\text{V}$$

1 % acima do V_REF no fim de curso — o último 1 % do potenciômetro satura, aceitável (o curso útil do amortecedor nunca chega ao batente do sensor). $R_{Th} = 8\,\text{k}\Omega$.

### 3.3 Divisor 38 k / 12 k (→ ADS131M08, pedal 0–5 V)

$$\alpha = \frac{12}{50} = 0{,}24 \quad\Rightarrow\quad 5{,}0\,\text{V} \to 1{,}20\,\text{V}$$

38 k não é E24; usar 39 k (0,235) ou 36 k + 2 k.

### 3.4 Impedância de fonte e o capacitor

Um SAR (MCP3208) carrega o capacitor interno de amostragem (~20 pF) a cada conversão. Com $R_{Th} = 8{,}8\,\text{k}\Omega$ a constante de tempo seria 0,18 µs — ok para 2 MHz de clock, mas o **capacitor externo de 100 nF** faz melhor: ele é 5000× o capacitor interno e entrega a carga instantaneamente. Sem ele, o divisor de alta impedância dá leitura que depende do canal lido antes (*crosstalk* de multiplexador).

O ADS131M08 é delta-sigma com entrada chaveada; o datasheet pede impedância de fonte baixa ou capacitor de filtro. Mesmo 100 nF.

---

## 📉 4. Filtro RC anti-aliasing

$$f_c = \frac{1}{2\pi RC}$$

| Configuração | R | C | f_c | Onde |
| :--- | :---: | :---: | :---: | :--- |
| **Padrão** | 1 kΩ | 100 nF | **1591,5 Hz** | Todos os canais do MCP3208 (amortecedor a 200 Hz, ADXL a 200/100 Hz) |
| Amortecida | 3,3 kΩ | 100 nF | 482,3 Hz | Canais lentos com ruído de bomba: NTCs, `lv_battery_voltage` |
| Divisor + C | 8,8 kΩ (Thévenin) | 100 nF | 181 Hz | O que o divisor 33k/12k **já faz sozinho** com o capacitor de 100 nF — para APPS/BSE a 100 Hz está no limite: usar **22 nF** (822 Hz) nesses canais |

Regra: $f_c \ge 5\times$ a banda útil do sinal e $\le 1/4$ da taxa de amostragem do ADC (não da taxa do frame CAN). O ADS131M08 amostra a 1 kSPS e decima; o MCP3208 é lido a 200 Hz, então o RC de 1,6 kHz **não** é anti-aliasing para 200 Hz. Recomendação para a Fase 1: ler o MCP3208 a 800 Hz e enviar a média de 4 amostras a 200 Hz (sobreamostragem ×4); a leitura de 5 canais leva ~50 µs, cabe com folga. Documentado para não se enganar.

---

## 🔬 5. INA333 para a célula de torque

$$G = 1 + \frac{100\,\text{k}\Omega}{R_G} \quad\Rightarrow\quad R_G = 499\,\Omega \Rightarrow G = 201$$

* Ponte completa, excitação 3,3 V, saída ±5 mV em ±25 Nm → ±1,0 V na saída do INA333.
* **REF = 1,65 V** (V_DD/2: divisor 10 k/10 k + buffer, ou o próprio pino REF do INA333 alimentado por um *op-amp* seguidor). O INA333 é alimentado com 3,3 V *single-supply*: a saída **não desce abaixo de ~0,05 V**, então a referência precisa ficar no meio da escala. Saída: 0,65–2,65 V.
* Ligar **AINP = saída do INA333, AINN = REF (1,65 V)**. O ADS131M08 mede a diferença: **±1,0 V, 83 % do FSR de ±1,2 V**. As entradas absolutas (0,65–2,65 V) ficam dentro de AGND…AVDD. Ganho e offset da célula vão para o NVS do VCU.
* Por que não *single-ended*: ligando só AINP, o ADC veria 0,65–2,65 V contra AGND e saturaria em 1,2 V. O ADS131M08 é diferencial — usar isso.
* A versão anterior tinha R_G = 390 Ω (G = 257) e mandava a saída *single-ended* para "um ADC de 0–3,3 V". Com o ADS131M08 isso satura.

---

## 🔗 Próxima Leitura
* [[🔵 No PCU (Controle, Powertrain, APS, Freios e Direcao)|🔵 Onde cada canal entra no VCU]]
* [[🟢 No VDN-Front (ESP32 Heltec V3, Entradas Digitais e SPI)|🟢 Canais do MCP3208 nos VDNs]]
* [[🛑 Transdutores de Pressao Hidraulica e APS Duplo Redundante|🛑 APPS em dois ADCs]]
* [[🔌 Guia Pratico de Conexao de Sensores (Analogicos, Digitais, I2C, SPI e Niveis Logicos)|🔌 Guia de conexão]]
