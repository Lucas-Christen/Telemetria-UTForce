---
tipo: nota-tecnica
area: faculdade
projeto: "JetBot ROS 2 — Formula Driverless Platform"
data_atualizacao: 2026-09-13
modulo: "02 - Simulacao & Visualizacao"
documento: "Cenas e Mundos de Simulação (Worlds SDF)"
autor: "Lucas Fernandes Christen"
tags:
  - mundos-sdf
  - cenarios
  - monaco
  - labirinto
  - simulacao
---

# 👁️ Cenas e Mundos de Simulação (Worlds SDF)

> Catálogo e especificação física dos cenários virtuais de teste disponíveis no diretório `gazebo_gz/worlds/`.

---

## 🧭 Visão Comparativa dos Mundos

A plataforma disponibiliza 6 ambientes distintos para cobrir desde a calibração inicial de sensores até corridas autônomas em alta velocidade:

| Arquivo de Mundo | Extensão / Dimensões | Tipo de Desafio | Sensores Principais Testados |
| :--- | :---: | :--- | :--- |
| **`jetbot_empty.sdf`** | $200 \times 200\text{ m}$ livre | Calibração de odometria, teste de resposta a degrau de velocidade e teleoperação pura. | Odometria (`/jetbot/odom`), Câmera. |
| **`track.sdf`** | $\approx 25\text{ m}$ (Linha central) | Circuito fechado orgânico com curvas de raio médio e aberturas controladas nas paredes. | LiDAR 360°, PCA de paredes, Perfil de velocidade AIMD. |
| **`track2.sdf`** | $\approx 35\text{ m}$ | Circuito técnico com chicanes rápidas e curvas em "S" consecutivas. | Frenagem antecipada, Follow-the-Gap em transições de rumo. |
| **`monaco.sdf`** | $\approx 60\text{ m}$ ($H = 0.6\text{ m}$) | Réplica do circuito de rua de Mônaco com rampas 3D de subida e descida. | Dinâmica de suspensão, odometria em rampa, curvas cegas de 180°. |
| **`maze.sdf`** | Grade $15 \times 15\text{ m}$ | Labirinto de corredores estreitos com esquinas fechadas de $90^\circ$. | Detecção de becos sem saída, manobra de ré reflexiva. |
| **`maze_obstacles.sdf`**| Grade $15 \times 15\text{ m}$ | Labirinto com cilindros e caixas obstruindo parcialmente o corredor central. | Desvio de obstáculos frontal com Follow-the-Gap ativo. |

---

## 🔬 Análise dos Mundos Chave

### 1. `track.sdf` & `track2.sdf` — O Banco de Provas Driverless
* **Física:** Motor DART com passo de tempo de integração $\Delta t = 1\text{ ms}$ (`max_step_size="0.001"`), garantindo estabilidade nas forças de contato entre as rodas e o asfalto.
* **Geometria:** Paredes contínuas de $0.3\text{ m}$ de altura com materiais de cores contrastantes para facilitar inspeção no RViz.
* **Linha de Partida:** Modelo embutido `start_line` posicionado na origem $(0, 0, 0.001)$ com caixa branca de $0.05\text{ m}$ de espessura atravessando a largura da pista.
* **Spawn do Robô:** O modelo `<uri>model://jetbot</uri>` é inserido automaticamente a $5\text{ mm}$ do chão na origem, alinhado em $+X$.

### 2. `monaco.sdf` — Réplica de Autódromo Internacional
* Utiliza as malhas 3D exportadas em `gazebo_gz/models/monaco_track/meshes/`:
  * `road.stl`: Pavimento asfáltico completo com inclinações e desníveis.
  * `wall_L.stl` e `wall_R.stl`: Guard-rails externos e internos modelados como corpos rígidos estáticos de colisão.
* Excelente para validação de endurance e teste de aquecimento térmico simulado.

### 3. `maze.sdf` — Teste de Sobrevivência e Cornering
* Se o algoritmo falhar ao contornar uma curva em $90^\circ$ e bater de frente contra uma parede a menos de $0.22\text{ m}$, o nó `race_follower.py` detecta a colisão iminente e dispara o protocolo de emergência:
  1. Desacelera a zero imediatamente.
  2. Engata ré suave ($v = -0.15\text{ m/s}$) por $0.8\text{ segundos}$.
  3. Gira as rodas no sentido do lado com maior folga residual.
  4. Retoma a navegação para frente.

---

## ⚡ Como Alternar Entre os Mundos

Basta passar o nome do arquivo através do argumento `world`:

```bash
# Executar no labirinto com obstáculos:
ros2 launch gazebo_gz/launch/jetbot.launch.py world:=maze_obstacles.sdf

# Executar na réplica de Mônaco:
ros2 launch gazebo_gz/launch/jetbot.launch.py world:=monaco.sdf race:=true
```

---

## 🔗 Próxima Leitura
* [[🏎️ Controlador de Corrida Autonoma (race_follower.py)|🏎️ Detalhes matemáticos do algoritmo de direção autônoma]]
* [[📈 Aprendizado Espacial de Velocidade por Trecho (AIMD Profiling)|📈 Como o robô memoriza a pista e anda mais rápido a cada volta]]
