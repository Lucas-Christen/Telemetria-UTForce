---
tipo: nota-tecnica
area: faculdade
projeto: "Telemetria UTForce E-Racing"
status: producao
descricao: "Auditoria de precisão técnica da documentação de telemetria (software desktop e hardware embarcado)"
data_atualizacao: 2026-09-14
tags:
  - telemetria
  - fsae
  - auditoria
  - revisao-tecnica
  - esp32
  - can-bus
---

# 🔍 Auditoria Técnica da Documentação de Telemetria

> Revisão de precisão das duas árvores de telemetria da UTForce: o **software desktop** (`04`) e o **hardware embarcado** (`06`). Cada afirmação abaixo foi verificada contra o código-fonte real, contra as outras notas do cofre ou contra o datasheet do componente.

**Data da auditoria:** 2026-09-14
**Método:** comparação linha a linha da documentação contra o repositório `Telemetria-Christen-UTFORCE`, checagem cruzada entre notas e conferência de pinagem contra a documentação oficial da Espressif e da Heltec.

---

## 📊 Veredito por documento

| Documento | Veredito |
| :--- | :--- |
| 📊 Dicionário dos 70+ Sensores | ✅ **Exato.** 77/77 canais, faixas e tipos conferem com o código. |
| 🚀 Setup, Dependências e Execução | ⚠️ Uma afirmação falsa sobre numpy. |
| 📦 Empacotamento PyInstaller | ✅ Confere com `main.spec`. |
| 🔄 Simulador Multi-thread e Pipeline | ❌ Descreve arquitetura que não é a do código. |
| 🏎️ Status Visual e Alertas Térmicos | ✅ Limiares conferem. Omite uma terceira fonte de dados. |
| ⚡ Circuitos Condicionadores | ✅ **Toda a matemática confere.** |
| 📡 Módulo LoRa SX1262 | ⚠️ Bitrate e tempo no ar corretos; contagem de bytes errada. |
| 📊 Matriz de Mensagens CAN | ⚠️ Escalas e larguras corretas; taxas conflitam com outra nota. |
| 📊 Dicionário de Canais e Taxas (Hz) | ❌ Contradiz a Matriz CAN em quase todos os canais. |
| 📌 Mapa Definitivo de Pinos do ESP32 | ❌ Documenta o chip errado para 3 dos 4 nós. |
| 🔵 Nó PCU | ❌ Pino analógico impossível na placa declarada. |
| 🟢 Nó VDN-Front | ❌ Conflito de pinos com o rádio embutido da placa. |
| 🟣 Nó INU | ❌ GPIO 34 atribuído a duas funções na mesma nota. |

---

## ✅ 1. O que está correto — e é mérito real

Antes dos defeitos, vale registrar o que passou em verificação rigorosa. Esta parte da documentação é melhor do que a média do que se vê em projetos de Fórmula SAE.

### 1.1 O dicionário de sensores é exato

Comparei as 77 chaves de `data/data_simulator.py` contra as 77 linhas das tabelas da nota:

* **Zero** canais documentados que não existem no código.
* **Zero** canais no código que faltam na documentação.
* **Zero** divergências de faixa numérica (`random.uniform(a, b)` bate com a coluna "Faixa Típica" em todos os casos).
* **Zero** divergências de tipo (`Float` / `Inteiro` / `Binário` conferem com `uniform` / `randint` / `choice`).

Documentação de catálogo com 100% de aderência ao código é raro. Esta nota pode ser usada como fonte de verdade sem ressalva.

### 1.2 A matemática dos circuitos condicionadores confere

Refiz todas as contas da nota:

* Divisor $\alpha = 12/(33+12) = 0{,}2667$ ✅
* $10{,}0\text{ V} \times 0{,}2667 = 2{,}67\text{ V}$ ✅
* $4{,}5\text{ V} \times 0{,}2667 = 1{,}20\text{ V}$ ✅
* Impedância de Thévenin $= 33\text{k} \parallel 12\text{k} = 8{,}8\text{ k}\Omega$ ✅
* Filtro RC $1\text{k}/100\text{nF} \rightarrow 1591{,}5\text{ Hz}$ ✅
* Filtro RC $3{,}3\text{k}/100\text{nF} \rightarrow 482{,}3\text{ Hz}$ ✅

### 1.3 O dimensionamento do LoRa está correto

* **Bitrate declarado: 21,8 kbps.** Confere. Para SF7, BW 500 kHz, CR 4/5: $7 \times \frac{500000}{2^7} \times \frac{4}{5} = 21\,875\text{ bps}$.
* **Tempo no ar < 25 ms.** Confere. Para 48 bytes nessa configuração o tempo no ar é de ≈ 24,4 ms, o que cabe folgadamente na janela de 100 ms de um envio a 10 Hz (≈ 24% de ciclo de trabalho).

A conclusão de engenharia está certa, mesmo com o erro de contagem de bytes apontado adiante.

### 1.4 A matriz CAN tem escalas bem dimensionadas

Verifiquei cada campo contra a largura do inteiro escolhido. **Nenhum estoura.** Alguns exemplos:

| Campo | Tipo | Escala | Contagens necessárias | Limite do tipo |
| :--- | :---: | :---: | :---: | :---: |
| `Wheel_Speed_FL` | uint16 | 0,01 | 30 000 | 65 535 ✅ |
| `Yaw_Rate` | int16 | 0,01 | ±30 000 | ±32 767 ✅ |
| `Latitude` | int32 | $10^{-7}$ | ±900 000 000 | ±2 147 483 647 ✅ |
| `Tyre_Temp_FL` | uint16 | 0,1 / offset −40 | 1 900 | 65 535 ✅ |

O `Yaw_Rate` é o mais apertado, com 92% da faixa do `int16` ocupada — funciona, mas não sobra margem se um dia quiserem medir além de ±300 °/s.

### 1.5 A carga do barramento CAN é confortável

A documentação não calcula isso, então calculei. Somando os frames definidos, com a estimativa de pior caso de $55 + 10 \times \text{DLC}$ bits por frame padrão:

| Frame | Taxa | DLC | Bits/s |
| :--- | :---: | :---: | ---: |
| `0x100` | 100 Hz | 8 | 13 500 |
| `0x101` | 50 Hz | 6 | 5 750 |
| `0x200` / `0x220` | 50 Hz | 8 | 13 500 |
| `0x201` / `0x221` | 10 Hz | 6 | 2 300 |
| `0x500` | 50 Hz | 8 | 6 750 |
| `0x501` | 10 Hz | 8 | 1 350 |
| **Total** | | | **≈ 43,2 kbps** |

Isso é **8,6% de ocupação** de um barramento de 500 kbps. Há margem enorme para o inversor, o BMS e o Raspberry Pi. A escolha de 500 kbps está bem justificada.

### 1.6 A hierarquia de prioridade dos IDs está correta

A premissa "menor ID = maior prioridade de arbitragem" está certa, e a alocação é coerente: circuito de segurança e BMS em `0x050~0x09F` (a faixa mais prioritária), telemetria de conforto nas faixas altas. É exatamente como se projeta uma rede veicular.

---

## ❌ 2. Defeitos no software desktop

### 2.1 Existem dois simuladores rodando ao mesmo tempo — CRÍTICO

Este é o achado mais sério da auditoria, e a documentação não só o omite como descreve o oposto.

O diagrama de sequência da nota 🔄 mostra um caminho único:

```
DataSimulator → DataProcessor → MainWindow
```

O código real instancia **dois** `DataSimulator` independentes:

**Instância 1** — dentro de `gui/main_window.py`, linhas 431–433:

```python
self.data_simulator = DataSimulator()
self.data_simulator.data_generated.connect(self.update_graphs_with_data)
self.data_simulator.start()
```

**Instância 2** — dentro de `main.py`, linhas 84–94:

```python
simulator = DataSimulator()
processor = DataProcessor()
processor.data_updated.connect(window.update_graphs_with_data)
simulator.data_generated.connect(processor.process_data)
simulator.start()
```

Consequências práticas:

1. **Duas threads geram números aleatórios independentes**, ambas a 10 Hz.
2. `update_graphs_with_data` é chamado **20 vezes por segundo**, não 10, alternando entre duas séries sem nenhuma relação entre si. O gráfico plotado é a intercalação de dois ruídos distintos.
3. A instância criada na `MainWindow` **conecta direto na GUI**, pulando o `DataProcessor`. Ou seja, metade dos dados nunca passa pela camada que a documentação apresenta como "camada de negócio e filtro".
4. O buffer de 100 pontos enche em 5 segundos em vez de 10.

> [!danger] Efeito sobre a documentação
> O diagrama de sequência da nota 🔄 e o diagrama de arquitetura da nota 📡 descrevem um sistema que não existe. Não é imprecisão de detalhe: é a topologia central do programa.

### 2.2 O `DataProcessor` não processa nada

A nota 🔄 apresenta este trecho como sendo o arquivo:

```python
def process_data(self, raw_data):
    """
    Ponto de extensão para processamento, calibração,
    conversão de unidades CAN e filtros passa-baixa.
    """
```

O arquivo real diz:

```python
def process_data(self, raw_data):
    """
    Processa os dados brutos e emite um sinal com os dados processados.
    """
    # Aqui você pode adicionar lógica de processamento dos dados
    # Para simplificar, vamos emitir os dados brutos diretamente
    processed_data = raw_data
```

A docstring foi **reescrita na documentação** para soar mais projetada do que é. O bloco está apresentado como citação do código-fonte, mas não é o código-fonte. Esse é o tipo de divergência mais perigoso, porque o leitor confia num bloco de código citado.

### 2.3 A afirmação sobre numpy é falsa

A nota 🚀 afirma:

> `numpy` é puxado automaticamente como dependência transitiva do `pyqtgraph`, **sendo utilizado diretamente no módulo de interpolação e comparação de voltas** (`comparison_view.py`).

`comparison_view.py` tem `import numpy as np` na linha 12, e **nenhuma outra ocorrência de `np.` no arquivo inteiro** — nem no resto do repositório. É um import morto.

A interpolação linear é escrita à mão, em Python puro, com varredura sequencial:

```python
for i in range(len(x_data) - 1):
    if x_data[i] <= x <= x_data[i + 1]:
        ...
        return y0 + (y1 - y0) * (x - x0) / (x1 - x0)
```

A matemática está correta (interpolação linear canônica, com proteção contra divisão por zero). O que é falso é a atribuição a numpy. Como isso roda a cada movimento do mouse sobre o gráfico, uma busca binária ou `np.interp` seria a melhoria óbvia — e aí o import deixaria de ser morto.

### 2.4 Existe uma terceira fonte de dados não documentada

`gui/car_monitoring_view.py` mantém o próprio `QTimer` e o próprio gerador aleatório, totalmente independente do `DataSimulator`:

```python
self.timer.start(500)  # Atualizar a cada 1 segundo
```

Dois problemas: a tela de monitoramento do carro não consome a telemetria do sistema, e o comentário diz "1 segundo" enquanto o código usa 500 ms.

### 2.5 O conjunto de canais é de carro a combustão

O README define a equipe como **Fórmula SAE Elétrica**, e o dicionário de sensores documenta corretamente `SOC`, `SOH`, tensão de 300 V e corrente de tração. Mas convivem com eles:

`ecu_fuel_pressure`, `ecu_fuel_pump`, `ecu_fuel_temp`, `ecu_fuel_total`, `tank_fuel`, `tank_fuel_used`, `fuel_economy`, `ecu_lambida_1/2` (sonda lambda de escapamento), `ecu_oil_pressure`, `ecu_oil_temp`, `ecu_oil_lamp`, `ecu_airbox_temp`, `ecu_gear` de 1 a 8, `ecu_rpm` até 12 000, `ecu_syncro` (sincronismo de virabrequim).

E em `car_monitoring_view.py` há um componente literalmente chamado `"Combustion Engine"` com limiar térmico de 85–110 °C, ao lado de `"Eletric Engine"`.

Isso tem cara de lista de canais herdada de uma ECU de combustão (padrão MoTeC/Pi), não derivada do carro real. Não é erro de documentação — a documentação descreve fielmente o que existe. É uma observação sobre o **produto**: metade do catálogo não corresponde ao veículo.

### 2.6 Inconsistências menores de nomenclatura no código

| Problema | Onde | Por que importa |
| :--- | :--- | :--- |
| `"energy used"` com **espaço** | `data_simulator.py` | Único canal de 77 que foge do snake_case. Quebra qualquer acesso por atributo e complica persistência de layout. |
| `SOC` / `SOH` em maiúsculas | `data_simulator.py` | Únicos dois canais não-minúsculos. |
| `ecu_lambida_1` / `_2` | `data_simulator.py` | Grafia incorreta de *lambda*. A documentação replica o erro fielmente — o que é o comportamento certo para um dicionário de chaves, mas vale corrigir nos dois lugares de uma vez. |
| `"Eletric Engine"` | `car_monitoring_view.py` | Grafia incorreta de *Electric*. |
| Versão `v1.0.0 (02/01/2024)` | `README.md` | O mesmo README diz que o projeto foi criado em **02/01/2025**, e `main.py` diz "Última atualização: 02/01/2024". Três datas conflitantes dentro do próprio repositório. |

---

## ❌ 3. Defeitos no hardware embarcado

Esta árvore não tem código para comparar — é documentação de projeto. Verifiquei então a consistência interna entre as notas e a aderência aos datasheets.

### 3.1 O mapa de pinos documenta o chip errado — CRÍTICO E SISTÊMICO

A nota 📌 **Mapa Definitivo de Pinos do ESP32** descreve, corretamente, o **ESP32 clássico** (ESP32-D0WD / WROOM-32):

* GPIO 34, 35, 36 (VP), 39 (VN) como entrada-apenas
* ADC1 = GPIO 32, 33, 34, 35, 36, 39
* ADC2 = GPIO 0, 2, 4, 12, 13, 14, 15, 25, 26, 27
* Conflito ADC2 × Wi-Fi
* Cinco *strapping pins*

Tudo isso é verdade **para o ESP32 clássico**. Mas as notas dos nós declaram:

| Nó | Placa declarada | Chip real |
| :--- | :--- | :--- |
| PCU | Heltec WiFi LoRa 32 V3 | **ESP32-S3** |
| VDN-Front | Heltec WiFi LoRa 32 V3 | **ESP32-S3** |
| VDN-Rear | Heltec WiFi LoRa 32 V3 | **ESP32-S3** |
| INU | LilyGO T-Beam | ESP32 clássico ✅ |

No **ESP32-S3**, segundo a Espressif:

* **ADC1 = GPIO 1 a 10**
* **ADC2 = GPIO 11 a 20**
* Não existem pinos "VP/VN"
* GPIO 34/35/36/39 **não são entrada-apenas nem ADC**
* Os *strapping pins* são outros (GPIO 0, 3, 45, 46)

Ou seja: a "Regra de Ouro" da equipe — *"100% dos sensores analógicos DEVEM ser conectados aos pinos do ADC1: GPIO 36, 39, 34, 35"* — está **correta para o T-Beam e errada para os três nós Heltec**. E é justamente onde estão os sensores analógicos do carro.

### 3.2 O PCU usa um pino que não é ADC e já tem dono

Consequência direta do item anterior. A nota 🔵 do PCU atribui:

> Potenciômetro Pad_Reg · Analógico 0~3,3 V · `GPIO 36` (Entrada analógica)

Na Heltec V3 (ESP32-S3), **GPIO 36 é o controle de Vext** — o pino que liga e desliga a alimentação dos periféricos externos da placa. E no ESP32-S3 ele não é canal de ADC de qualquer forma.

Usar GPIO 36 como entrada analógica ali resulta em leitura inválida **e** em risco de desligar a alimentação externa da própria placa.

### 3.3 O VDN-Front colide com o rádio embutido da placa — CRÍTICO

A nota 🟢 atribui o barramento SPI do ADC MCP3208 assim:

| Sinal na nota | GPIO | O que esse pino é na Heltec V3 |
| :--- | :---: | :--- |
| SPI CLK | 10 | **SX1262 MOSI** |
| SPI MISO | 11 | **SX1262 MISO** |
| SPI MOSI | 12 | **SX1262 RST** |
| SPI CS | 13 | **SX1262 BUSY** |

Os quatro pinos do MCP3208 caem exatamente em cima do rádio LoRa SX1262 soldado na placa. O SX1262 não é opcional: ele está fisicamente ligado a esses pads.

O caso mais grave é o **GPIO 12 = RST do rádio**. Usá-lo como MOSI mantém o rádio em reset ou o chaveia aleatoriamente a cada conversão do ADC.

> [!warning] Isso inviabiliza a arquitetura como está
> Se o VDN-Front precisa ler o MCP3208 **e** a equipe usa o LoRa embutido em algum nó Heltec, o SPI do ADC tem de migrar para outros GPIOs livres do S3 (2 a 7 ou 17 a 21, por exemplo), ou o nó tem de sair da Heltec V3 para uma placa sem rádio.

### 3.4 O nó INU atribui GPIO 34 a duas funções

Dentro da **mesma nota** 🟣:

* No diagrama Mermaid: `PPS -->|"Interrupção Externa GPIO 34"| TBeam_Core`
* Na tabela de pinagem: `GPS UART RX | NEO-M8N TX | GPIO 34` e `GPS PPS | NEO-M8N TimePulse | GPIO 37`

O diagrama e a tabela discordam. Um dos dois está errado e não há como saber qual sem consultar a placa.

Ponto adicional a verificar na bancada: **GPIO 37 não é exposto na maioria dos módulos ESP32** — costuma ficar reservado à PSRAM. Vale confirmar se o T-Beam de vocês realmente disponibiliza esse pino antes de fechar a PCB.

### 3.5 Duas notas discordam sobre as taxas de amostragem — CRÍTICO

O 📊 **Dicionário de Canais e Taxas (Hz)** e a 📊 **Matriz de Mensagens CAN** especificam frequências diferentes para os mesmos canais:

| Canal | Dicionário de Canais | Matriz CAN | Fator |
| :--- | :---: | :---: | :---: |
| `Throttle Pos` | 20 Hz | `0x100` @ 100 Hz | 5× |
| `Brake Press F/R` | 10 Hz | `0x100` @ 100 Hz | 10× |
| `Steered Angle` | 20 Hz | `0x101` @ 50 Hz | 2,5× |
| `Steer Torque` | 10 Hz | `0x101` @ 50 Hz | 5× |
| `Wheel Speed` | 10 Hz | `0x200` @ 50 Hz | 5× |
| `Ride Height` | 10 Hz | `0x200` @ 50 Hz | 5× |
| `Tyre Temp` | 2 Hz | `0x201` @ 10 Hz | 5× |
| `G Force Lat/Long/Vert` | 20 Hz | `0x500` @ 50 Hz | 2,5× |
| `Hub az` | 10 Hz | `0x201` @ 10 Hz | ✅ |
| `GPS Lat/Lon` | 10 Hz | `0x501` @ 10 Hz | ✅ |

E há ainda um **terceiro** conjunto: a tabela de faixas de ID, no topo da própria Matriz CAN, diz "PCU: 50~100 Hz", "VDN: 20~50 Hz", "INU: 10~50 Hz".

Há também um problema **estrutural**, não apenas numérico: o frame `0x100` empacota `Throttle_Pos` (que o dicionário quer a 20 Hz) junto com `Brake_Press_Front/Rear` (que o dicionário quer a 10 Hz) num único frame a 100 Hz. **Em CAN, um frame tem uma taxa só.** As taxas por canal do dicionário são inatingíveis com esse agrupamento de payload — seria preciso separar os canais em frames distintos.

Enquanto isso não for resolvido, o firmware não tem especificação: dois documentos oficiais mandam coisas diferentes.

### 3.6 O acelerômetro escolhido não cobre a faixa especificada

* O **Dicionário de Canais** define a fonte de `FL/FR/RL/RR hub az` como **ADXL335**.
* A **Matriz CAN** define `Hub_az_FL` como `int16` × 0,01, faixa **±15,00 g**.

O ADXL335 é um acelerômetro analógico de **±3 g**. Ele não consegue medir ±15 g: satura em um quinto da faixa declarada no protocolo.

E a faixa de ±15 g da matriz CAN é a que está fisicamente correta — acelerações verticais de massa não suspensa em pista passam de 3 g com facilidade ao pegar zebra. Portanto o erro está na **escolha do sensor**, não na especificação do protocolo. Um ADXL377 (±200 g) ou um H3LIS331DL (±100/200/400 g) seriam adequados.

### 3.7 O pacote LoRa tem 41 bytes, não 48

A nota 📡 afirma "exatos 48 bytes" três vezes. Somei os campos da `struct` — que está sob `#pragma pack(push, 1)`, logo sem padding algum:

| | |
| :--- | ---: |
| 24 campos declarados | |
| **Soma real** | **41 bytes** |
| Declarado na nota | 48 bytes |
| Diferença | **−7 bytes** |

A conclusão de engenharia **sobrevive** ao erro: com 41 bytes o tempo no ar cai para ≈ 21,8 ms, ainda mais folgado que os 24,4 ms de 48 bytes. Mas o número citado está errado, e os 7 bytes livres são espaço útil que a equipe não sabe que tem.

### 3.8 O VDN-Rear não tem tabela de pinagem própria

A nota do VDN-Rear declara reaproveitar a mesma PCB e o mesmo microcontrolador do VDN-Front, o que é uma decisão de projeto boa e bem justificada (peças sobressalentes intercambiáveis nos boxes). Mas ela não traz tabela de pinos própria nem remete explicitamente à do VDN-Front. Quem for montar o nó traseiro precisa deduzir.

---

## ❓ 4. O que não foi possível verificar

Registro para não dar falsa sensação de cobertura total:

* **Não existe repositório de firmware embarcado** nas pastas locais. Toda a árvore `06` é documentação de projeto sem código correspondente — os defeitos acima são de consistência interna e de datasheet, não de aderência a uma implementação.
* **O Manual Mestre de Telemetria Veicular** (125 KB) não foi auditado linha a linha. Como é um compilado das notas modulares, é provável que replique os mesmos defeitos.
* **As notas de Blackbox (Raspberry Pi) e de migração para ROS 2** ficaram fora deste passe.
* **Consumo de RAM de 80–120 MB** afirmado na nota 🔄: não verificável sem executar o programa.
* **Disponibilidade física do GPIO 37** no T-Beam: exige conferência na placa.

---

## 🎯 5. Ações sugeridas, por prioridade

### Prioridade 1 — Impedem o sistema de funcionar

1. **Remover um dos dois `DataSimulator`.** Manter o de `main.py` (que passa pelo `DataProcessor`) e apagar as linhas 431–433 de `main_window.py`, junto com a chamada em `closeEvent`. Sem isso, todo gráfico do software mostra dado corrompido.
2. **Remapear o SPI do MCP3208 no VDN-Front.** Os GPIOs 10–13 pertencem ao SX1262 da Heltec V3.
3. **Escolher uma taxa por canal** entre o Dicionário de Canais e a Matriz CAN, e reagrupar os payloads para que cada frame tenha uma taxa única.

### Prioridade 2 — Documentação que induz ao erro

4. **Dividir o mapa de pinos em dois documentos:** um para ESP32 clássico (T-Beam/INU) e outro para ESP32-S3 (os três nós Heltec). Hoje há uma única "Regra de Ouro" que é inválida para 3 dos 4 nós.
5. **Corrigir o `GPIO 36` do PCU** para um canal de ADC1 do S3 (GPIO 1 a 10).
6. **Resolver o GPIO 34 do INU** — diagrama e tabela discordam.
7. **Corrigir a docstring citada do `DataProcessor`** para o texto real do arquivo, e atualizar os dois diagramas de arquitetura para refletir o caminho duplo enquanto ele existir.

### Prioridade 3 — Precisão

8. **Trocar o ADXL335** por um acelerômetro de faixa compatível, ou reduzir a faixa do `Hub_az` na matriz CAN para ±3 g.
9. **Corrigir "48 bytes" para 41 bytes** na nota do LoRa, e decidir o que fazer com os 7 bytes sobrando.
10. **Remover a afirmação sobre numpy** da nota de setup, ou de fato usar `np.interp` na interpolação.
11. **Padronizar `"energy used"`, `SOC`, `SOH` e `ecu_lambida`** no código e na documentação ao mesmo tempo.
12. **Unificar as datas** do README e do cabeçalho de `main.py`.
13. **Documentar o `QTimer` próprio** do `car_monitoring_view.py`, ou ligá-lo ao pipeline principal.

---

## 🔗 Documentos auditados

* [[📡 Software de Telemetria UTForce - Visao Geral]]
* [[📊 Dicionario Completo dos 70+ Sensores Automotivos]]
* [[🔄 Simulador Multi-thread, Qt Signals e Pipeline de Dados]]
* [[🚀 Setup, Dependencias e Execucao Local]]
* [[📡 Hardware de Telemetria Embarcada - Visao Geral]]
* [[📌 Mapa Definitivo de Pinos do ESP32 (Pinout, Strapping Pins, ADC1 vs ADC2 e Regras de Ouro)]]
* [[📊 Matriz de Mensagens CAN e Dicionario de IDs (DBC Veicular)]]
* [[📊 Dicionario Completo de Canais, Fontes e Taxas de Amostragem (Hz)]]
* [[📡 Modulo LoRa SX1262, Bit-Packing Compacto e CRC16]]

## 📚 Fontes externas consultadas

* Espressif — [ADC do ESP32-S3: ADC1 = GPIO 1–10, ADC2 = GPIO 11–20](https://docs.espressif.com/projects/esp-idf/en/v4.4/esp32s3/api-reference/peripherals/adc.html)
* Heltec / ESPHome — [Heltec WiFi LoRa 32 V3: GPIO 36 = controle de Vext](https://devices.esphome.io/devices/heltec-wifi-lora-32-v3/)
* Pinagem do SX1262 na Heltec V3 — [NSS 8, SCK 9, MOSI 10, MISO 11, RST 12, BUSY 13, DIO1 14](https://github.com/JJJS777/Hello_World_RadioLib_Heltec-V3_SX1262)
