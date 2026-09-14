---
tipo: nota-tecnica
area: faculdade
projeto: "Telemetria UTForce E-Racing"
data_atualizacao: 2026-09-13
modulo: "03 - Interface & Telas"
documento: "Setup Dinâmico e Parâmetros de Suspensão"
autor: "Lucas Fernandes Christen"
tags:
  - setup
  - suspensao
  - aerodinamica
  - dinamica-veicular
  - fsae
---

# 🔧 Setup Dinâmico e Parâmetros de Suspensão

> Módulo de calibração geométrica, rigidez de suspensão, incidência aerodinâmica e algoritmo de identificação da melhor configuração (*Fastest Lap Setup*).

---

## 🎯 O Papel do Setup na Performance de Pista

Em uma competição de Fórmula SAE, o piloto só atinge o limite do protótipo se o acerto mecânico (*setup*) estiver perfeitamente casado com o asfalto, a temperatura da pista e a velocidade média do traçado.

O módulo [SetupView](file:///home/lucaschristen/Documentos/UTFPR/Telemetria-Christen-UTFORCE/gui/setup_view.py) atua como a planilha inteligente de calibração do engenheiro de chassis, permitindo correlacionar alterações de regulagem com os tempos de volta registrados.

---

## 📊 A Matriz dos 14 Parâmetros de Calibração

O software estrutura os dados em uma tabela `QTableWidget` com 14 colunas técnicas padronizadas:

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                             Configuração de Setup do Carro                                  │
├─────────┬──────────────┬─────────────┬───────────┬────────────┬─────────────┬───────────────┤
│ Setup   │ Rear Height  │ Front Height│ Rear Push │ Front Push │ Preload Spr │ Spring Height │
├─────────┼──────────────┼─────────────┼───────────┼────────────┼─────────────┼───────────────┤
│ Setup 1 │ 142.50 mm    │ 120.30 mm   │ 15.20 mm  │ 12.10 mm   │ 45.00 mm    │ 55.40 mm      │
│ Setup 2 │ 138.10 mm    │ 118.00 mm   │ 14.80 mm  │ 11.50 mm   │ 48.20 mm    │ 54.00 mm      │
└─────────┴──────────────┴─────────────┴───────────┴────────────┴─────────────┴───────────────┘
```

### Dicionário de Parâmetros de Setup:
1. **`Setup`**: Identificador da sessão ou volta de teste (ex: `Setup 1`, `Setup 2` ... `Setup 30`).
2. **`Rear Height` (mm)**: Altura estática de rodagem traseira (*Rear Ride Height*).
3. **`Front Height` (mm)**: Altura estática de rodagem dianteira (*Front Ride Height*).
4. **`Rear Push` (mm)**: Ajuste de comprimento da barra de acionamento do amortecedor traseiro (*Pushrod*).
5. **`Front Push` (mm)**: Ajuste de comprimento da haste do pushrod dianteiro.
6. **`Preload Springs` (mm)**: Pré-carga imposta sobre as molas helicoidais com o carro sem piloto.
7. **`Spring Height` (mm)**: Altura livre e curso útil de trabalho da mola.
8. **`Tire Calibration` (psi)**: Pressão de inflagem dos pneus calibrada a quente/frio.
9. **`Rake` (graus/adimensional)**: Diferença de inclinação longitudinal entre a traseira e a dianteira (fundamental para gerar sucção no assoalho difusor).
10. **`Wing Inclination` (graus)**: Ângulo de ataque das asas aerodinâmicas dianteira e traseira para geração de *downforce*.
11. **`Car Weight` (kg)**: Massa estática total aferida nas balanças de canto com piloto e fluidos.
12. **`Balance` (%)**: Distribuição percentual de peso longitudinal e equilíbrio entre sobresterço (*oversteer*) e subesterço (*understeer*).
13. **`Fuel in the Tank` (l)**: Volume de combustível ou peso do pack de baterias no instante do teste.
14. **`Lap Time (s)`**: Tempo final cronometrado da volta sob aquela configuração.

---

## 🏆 Algoritmo de Otimização do Setup Mais Rápido

O botão **"Comparar Setups"** executa uma varredura sobre toda a matriz de testes para encontrar a configuração ótima de menor tempo de volta:

```python
def compare_setups(self):
    if not self.setups:
        QMessageBox.warning(self, "Sem Dados", "Por favor, simule dados antes de comparar.")
        return

    # Otimização por mínimo do tempo de volta
    fastest_setup = min(self.setups, key=lambda x: x["Lap Time"])
    
    details = "\n".join([
        f"Setup: {fastest_setup['Setup']}",
        f"Lap Time: {fastest_setup['Lap Time']} s",
        f"Rear Height: {fastest_setup['Rear Height']} mm",
        f"Front Height: {fastest_setup['Front Height']} mm",
        f"Rear Push: {fastest_setup['Rear Push']} mm",
        f"Front Push: {fastest_setup['Front Push']} mm",
        f"Preload Springs: {fastest_setup['Preload Springs']} mm",
        f"Spring Height: {fastest_setup['Spring Height']} mm",
        f"Tire Calibration: {fastest_setup['Tire Calibration']} psi",
        f"Rake: {fastest_setup['Rake']}",
        f"Wing Inclination: {fastest_setup['Wing Inclination']} graus",
        f"Car Weight: {fastest_setup['Car Weight']} kg",
        f"Balance: {fastest_setup['Balance']}",
        f"Fuel in the Tank: {fastest_setup['Fuel in the Tank']} l",
    ])
    QMessageBox.information(self, "Setup Mais Rápido", f"O setup mais rápido foi:\n\n{details}")
```

O diálogo modal sintetiza instantaneamente qual combinação de geometria e carga aerodinâmica produziu a melhor performance para a equipe de mecânica replicar no carro antes das provas oficiais de piracicaba.

---

## 🔗 Próxima Leitura
* [[📡 Software de Telemetria UTForce - Visao Geral|Voltar para a Visão Geral da Telemetria]]
* [[🏎️ Chassi, Dinamica Veicular, Freios e Aerodinamica|Entender a física do chassi e suspensão da UTForce]]
