---
tipo: nota-tecnica
area: faculdade
projeto: "JetBot ROS 2 — Formula Driverless Platform"
data_atualizacao: 2026-09-13
modulo: "01 - Arquitetura, Ambiente & Setup"
documento: "Setup do Ambiente ROS 2 Jazzy, Gazebo Harmonic e Dependências"
autor: "Lucas Fernandes Christen"
tags:
  - ros2
  - ros2-jazzy
  - gazebo-harmonic
  - kubuntu
  - setup
  - simulacao
---

# 🚀 Setup do Ambiente ROS 2 Jazzy, Gazebo Harmonic e Dependências

> Guia de configuração do ecossistema de robótica em Kubuntu 24.04, migração arquitetural do Gazebo Classic para o Gazebo Harmonic e fluxo de execução ágil.

---

## 💻 Ambiente Operacional Homologado

* **Sistema Operacional:** Kubuntu 24.04 LTS (Kernel Linux 6.8+).
* **Distribuição de Robótica:** **ROS 2 Jazzy Jalisco** (Suporte oficial até 2029).
* **Simulador Físico:** **Gazebo Harmonic 8.11** (Executável nativo `gz`, interface moderna da Open Robotics).
* **Ponte de Mensagens:** `ros_gz_bridge` (Substitui os antigos plugins C++ compilados in-tree do Gazebo Classic).
* **Compilação e Dev Tools:** `colcon`, `rosdep`, `ament_cmake`, Python 3.12 com `rclpy`.

---

## ⚡ Contexto da Migração Técnica (Classic 11 $\to$ Harmonic 8)

O repositório upstream original (`dusty-nv/jetbot_ros`) foi desenvolvido na era do **ROS 2 Humble + Gazebo Classic 11**. Essa base de código antiga sofria dos seguintes gargalos fatais no Ubuntu 24.04:
1. **Descontinuação do Gazebo Classic:** O Gazebo 11 não possui pacotes oficiais nativos para Ubuntu 24.04 e encerrou seu ciclo de vida em 2025.
2. **Plugins Incompatíveis:** Diretivas legadas como `libgazebo_ros_diff_drive.so` e nós de spawn via serviço `/spawn_entity` não existem no novo Gazebo Sim (Harmonic).
3. **Inconsistência de Malhas 3D:** O modelo SDF original referenciava arquivos inexistentes como `Chassis_collision.dae` e `Wheel_simplified.stl`.

### A Refatoração de Lucas Christen:
Lucas reestruturou o pipeline criando o módulo [gazebo_gz/](file:///home/lucaschristen/Documentos/jetbot_ros-master/gazebo_gz):
* Criou um novo modelo [model.sdf](file:///home/lucaschristen/Documentos/jetbot_ros-master/gazebo_gz/models/jetbot/model.sdf) compatível com a especificação **SDF 1.9**.
* Substituiu plugins antigos pelos sistemas nativos do GZ: `gz::sim::systems::DiffDrive`, `gz::sim::systems::Sensors` e sensor `gpu_lidar`.
* Padronizou a rotação e escala ($0.001$, conversão milímetro $\to$ metro) das malhas oficiais presentes no repositório: `JetBot-v3-Chassis.dae` e `JetBot-v3-Wheel.stl`.
* Desenvolveu uma ponte declarativa via YAML com o `ros_gz_bridge`.

---

## 📦 Pacotes Necessários no Sistema Operacional

Caso seja necessário replicar o ambiente em uma nova máquina ou contêiner:

```bash
# 1. Atualizar repositórios do sistema
sudo apt update && sudo apt upgrade -y

# 2. Instalar a base do ROS 2 Jazzy Desktop e ferramentas de desenvolvimento
sudo apt install -y ros-jazzy-desktop ros-dev-tools

# 3. Instalar a integração com o Gazebo Harmonic e o simulador
sudo apt install -y ros-jazzy-ros-gz ros-jazzy-teleop-twist-keyboard

# 4. Configurar o sourcing automático do ROS no shell bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## 🚀 Fluxo de Inicialização Direto (Sem necessidade de `colcon build`)

Para maximizar a produtividade e eliminar o ciclo lento de compilação C++ durante ajustes de parâmetros de simulação, o launch principal [jetbot.launch.py](file:///home/lucaschristen/Documentos/jetbot_ros-master/gazebo_gz/launch/jetbot.launch.py) foi construído para **rodar diretamente a partir do caminho do arquivo**:

```bash
cd "/home/lucaschristen/Documentos/jetbot_ros-master"

# Carregar o ambiente do ROS
source /opt/ros/jazzy/setup.bash

# Rodar simulação padrão (Pista vazia + RViz2)
ros2 launch gazebo_gz/launch/jetbot.launch.py

# Rodar no circuito fechado gerado com o nó de corrida autônoma ativado
ros2 launch gazebo_gz/launch/jetbot.launch.py world:=track.sdf race:=true
```

---

## 🛑 Gerenciamento de Processos e Limpeza

Em simulações físicas, quando um processo é interrompido com `Ctrl + C`, instâncias em background do servidor Gazebo (`ruby`, `gz sim` ou `parameter_bridge`) podem continuar presas ocupando portas de rede e memória compartilhada.

Para garantir que a simulação seguinte inicie sem conflitos:

```bash
# Matar qualquer processo residual do Gazebo ou RViz antes de relançar
pkill -9 -f "gz sim" 2>/dev/null
pkill -9 -f "parameter_bridge" 2>/dev/null
pkill -9 -f "rviz2" 2>/dev/null
pkill -9 -f "race_follower" 2>/dev/null
```

---

## 🔗 Próxima Leitura
* [[📦 Modelo SDF 1.9 e Fisica DART do Robo|📦 Estrutura cinemática, inércias e sensores do robô]]
* [[🌐 Orquestrador de Launch Unificado (jetbot.launch.py)|🌐 O arquivo de launch e parâmetros de execução]]
