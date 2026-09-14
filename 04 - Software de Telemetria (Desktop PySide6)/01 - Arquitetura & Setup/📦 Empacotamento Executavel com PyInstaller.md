---
tipo: nota-tecnica
area: faculdade
projeto: "Telemetria UTForce E-Racing"
data_atualizacao: 2026-09-13
modulo: "01 - Arquitetura & Setup"
documento: "Empacotamento Executável com PyInstaller"
autor: "Lucas Fernandes Christen"
tags:
  - pyinstaller
  - deploy
  - devops
  - executavel
  - binario
---

# 📦 Empacotamento Executável com PyInstaller

> Engenharia de empacotamento, resolução dinâmica de caminhos em runtime via `_MEIPASS` e geração do binário autônomo para a bancada de testes e boxes de competição.

---

## 🎯 Objetivo do Empacotamento

Em ambiente de competição de Fórmula SAE (boxes e pista em Piracicaba/SP), os notebooks de engenharia e os computadores dos juízes de prova estática/dinâmica frequentemente não possuem interpretador Python, bibliotecas Qt ou compiladores instalados. 

O empacotamento via **PyInstaller** encapsula:
1. O interpretador Python congelado.
2. Todas as DLLs e shared libraries C++ do Qt 6 (`QtCore`, `QtGui`, `QtWidgets`).
3. Módulos compilados do `pyqtgraph` e `numpy`.
4. Os recursos estáticos do carro (especialmente o blueprint `assets/images/car_diagram.png`).
5. O binário final gerado em `dist/main.exe`.

---

## 📜 Análise do Arquivo de Especificação (`main.spec`)

O projeto utiliza um arquivo declarativo de compilação [main.spec](file:///home/lucaschristen/Documentos/UTFPR/Telemetria-Christen-UTFORCE/main.spec):

```python
# -*- mode: python ; coding: utf-8 -*-

a = Analysis(
    ['main.py'],
    pathex=[],
    binaries=[],
    datas=[('assets/images/car_diagram.png', 'assets/images')],
    hiddenimports=[],
    hookspath=[],
    hooksconfig={},
    runtime_hooks=[],
    excludes=[],
    noarchive=False,
    optimize=0,
)
pyz = PYZ(a.pure)

exe = EXE(
    pyz,
    a.scripts,
    a.binaries,
    a.datas,
    [],
    name='main',
    debug=False,
    bootloader_ignore_signals=False,
    strip=False,
    upx=True,
    upx_exclude=[],
    runtime_tmpdir=None,
    console=False,                  # Modo janela pura (sem terminal preto em background)
    disable_windowed_traceback=False,
    argv_emulation=False,
    target_arch=None,
    codesign_identity=None,
    entitlements_file=None,
)
```

### Decisões Técnicas de Compilação:
* **`datas=[('assets/images/car_diagram.png', 'assets/images')]`**: Garante que o diagrama vetorial/rasterizado do carro seja injetado no pacote e copiado para a pasta de runtime.
* **`console=False`**: Suprime a abertura da janela do prompt de comando do Windows, garantindo aparência profissional e limpa como software desktop corporativo.
* **`upx=True`**: Habilita o compressor executável Ultimate Packer for eXecutables (UPX), reduzindo significativamente o tamanho final do `.exe`.

---

## 🧩 O Padrão `resource_path` para Resolução de Ativos

Quando um script Python é congelado em um arquivo executável pelo PyInstaller, os arquivos embutidos são descompactados em um diretório temporário dinâmico armazenado na variável global `sys._MEIPASS`.

Para que o código funcione de forma idêntica tanto em desenvolvimento local quanto dentro do executável compilado, Lucas Christen implementou o helper `resource_path` em [car_monitoring_view.py](file:///home/lucaschristen/Documentos/UTFPR/Telemetria-Christen-UTFORCE/gui/car_monitoring_view.py):

```python
import os
import sys

def resource_path(relative_path):
    """
    Obtém o caminho absoluto do recurso, compatível com PyInstaller e modo dev.
    """
    try:
        # PyInstaller cria um diretório temporário e armazena o caminho em _MEIPASS
        base_path = sys._MEIPASS
    except AttributeError:
        # Modo interpretado padrão de desenvolvimento
        base_path = os.path.abspath(".")
    return os.path.join(base_path, relative_path)
```

### Uso no Carregamento do Diagrama do Veículo:
```python
image_path = resource_path("assets/images/car_diagram.png")
self.car_pixmap = QPixmap(image_path)
if self.car_pixmap.isNull():
    print(f"Erro ao carregar a imagem: {image_path}")
self.car_image.setPixmap(self.car_pixmap.scaled(800, 600))
```

---

## 🛠️ Procedimento de Compilação

Para compilar novamente o executável após alterações no código:

```bash
# 1. Ativar o ambiente com as dependências e o PyInstaller instalado
pip install pyinstaller

# 2. Executar a compilação utilizando o arquivo de especificação
pyinstaller main.spec --clean
```

### Estrutura de Artefatos Gerada:
* `build/main/`: Diretório com tabelas de conteúdo (`.toc`), dependências intermediárias compiladas e arquivos de log do processo.
* `dist/main.exe`: O executável final independente.
* `dist/graph_layout_config.json`: Arquivo de configuração de layout de gráficos que acompanha o binário para manter as preferências de visualização dos engenheiros entre sessões.

---

## 🔗 Próxima Leitura
* [[🔄 Simulador Multi-thread, Qt Signals e Pipeline de Dados|🔄 Entendendo a concorrência e os Qt Signals]]
* [[📈 Monitoramento em Tempo Real e Graficos Draggable (pyqtgraph)|📈 A interface gráfica de monitoramento em tempo real]]
