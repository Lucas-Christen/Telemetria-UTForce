---
tipo: nota-tecnica
area: faculdade
projeto: "Site UTForce"
data_atualizacao: 2026-09-13
topico: "Bancada de Avaliação Automatizada do Chatbot (eval-chatbot.mjs)"
tags:
  - faculdade/extensao
  - nlp
  - benchmarking
  - testes
  - avaliacao
---

# 🧪 Bancada de Avaliação & Métricas do Chatbot (`eval-chatbot.mjs`)

Especificação da suíte de teste e medição empírica do assistente virtual (`scripts/eval-chatbot.mjs`), garantindo que nenhuma alteração degrade a acurácia ou aumente alucinações.

---

## 🎯 1. Filosofia de Avaliação: Cobertura vs Precisão de Recusa

Em assistentes de sites institucionais, **otimizar apenas a taxa de acerto em perguntas conhecidas é uma armadilha clássica de NLP**:
* Um bot configurado com limiares muito baixos responde qualquer frase com confiança, mas **chuta errado em metade das perguntas fora do assunto**.
* **Num roteador de atendimento, resposta errada é infinitamente pior que "não entendi":** Quando o bot diz "não sei", o visitante reformula a pergunta ou clica em contato. Quando o bot inventa ou roteia errado, o usuário é induzido ao erro e perde a confiança na equipe.

Por isso, a bancada mede duas métricas complementares:
1. **Cobertura (*Recall*):** Das perguntas que a equipe **sabe** responder, quantas o bot acerta?
2. **Precisão de Recusa (*Refusal Precision*):** Das perguntas sem resposta cadastrada (intenções marcadas como `MISS`), quantas o bot **corretamente recusa com fallback**?

---

## 📊 2. Conjunto de Teste Reservado (105 Frases)

O arquivo `src/lib/chatbot/intents.seed.json` mantém um conjunto de **105 frases reservadas** (`eval` e `evalPersonas`):
* Cobre personas realistas: o estudante querendo entrar na equipe, o empresário querendo patrocinar, o visitante curioso, e testes adversariais com perguntas fora de escopo (ex: *"qual a cor do céu?"*, *"quem ganhou a copa?"*).

> [!CAUTION]
> **Regra Anti-Vazamento (Data Leakage):**
> As 105 frases de teste **nunca devem ser adicionadas como âncoras de treino**. Adicioná-las mascararia a capacidade de generalização do algoritmo e inflaria os números de forma artificial.

---

## 💻 3. Como Executar a Bancada

```bash
# Execução padrão (resumo consolidado de métricas)
npm run eval

# Execução detalhada (imprime cada frase com erro para investigação clínica)
node scripts/eval-chatbot.mjs --errors
```

---

## 📈 4. Resultados Históricos Medidos

Resultados comparativos medidos na evolução do assistente (Recuperação Pura vs Geração Baseada em Fatos):

| Métrica | Versão Inicial (Texto Fixo) | Versão Atual (NLG Factual) | Meta Mínima |
| :--- | :---: | :---: | :---: |
| **Acurácia Geral** | 84% | **84%** | $\ge 80\%$ |
| **Cobertura de Tópicos** | 82% | **82%** | $\ge 80\%$ |
| **Precisão de Recusas (`MISS`)** | 14/15 (93%) | **14/15 (93%)** | $\ge 90\%$ |
| **Respostas Dinâmicas Compostas**| 0% | **34%** | $\ge 30\%$ |

* **Interpretação:** A transição para o motor de NLG dinâmico manteve a altíssima precisão de recusas e a cobertura, mas transformou **34% das respostas em textos ricos e vivos** extraídos diretamente do `roster.json` (número de membros na área, coordenação correspondente e convite contextual).

---
* 🔗 Voltar para a [[🌐 Site UTForce - Visao Geral|Visão Geral do Site UTForce]]
* 🔗 Ver [[🌐 Contrato da API Serverless e Endpoints|Contrato da API Serverless]]
