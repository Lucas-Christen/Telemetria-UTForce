---
tipo: nota-tecnica
area: faculdade
projeto: "Site UTForce"
data_atualizacao: 2026-09-13
topico: "Similaridade Vetorial por N-Gramas, Ponderação IDF e Tolerância Levenshtein"
tags:
  - faculdade/extensao
  - nlp
  - matematica
  - tf-idf
  - levenshtein
  - algoritmos
---

# 📐 Similaridade por N-Gramas & Tolerância Levenshtein

Especificação matemática dos algoritmos de processamento de texto e similaridade vetorial (`similarity.js` e `text.js`), demonstrando a superioridade de métodos determinísticos leves sobre modelos pré-treinados pesados.

---

## 🧮 1. Vetorização por Trigramas de Caracteres ($N=3$)

Ao contrário de n-gramas de palavras, os **trigramas de caracteres** capturam radicais, sufixos e pequenas variações ortográficas sem exigir tabelas de lematização:
1. **Espaçamento de Borda:** A string normalizada é envolvida por espaços (`" texto "`) para que os trigramas capturem o início e o fim de cada palavra (ex: `" te"`, `"ext"`, `"to "`).
2. **Frequência de Termos:** Cada trigrama é computado em um mapa esparso de contagem.

---

## ⚖️ 2. Ponderação por Frequência Inversa de Documento (IDF)

> [!NOTE]
> **Por que trigramas puros falhavam?**
> No português, trigramas como `" de "`, `"ent"`, `"ção"`, `"que"` aparecem em quase toda frase. Sem ponderação, o cosseno media se duas frases eram escritas em português, e não se falavam do mesmo assunto.

O sistema aplica o cálculo de **IDF** sobre as âncoras da base:
$$IDF(\text{gram}) = \ln\left(1 + \frac{N_{\text{âncoras}}}{1 + DF(\text{gram})}\right)$$

* **Trigramas Ubíquos:** Recebem peso próximo a zero, sendo desconsiderados na comparação.
* **Trigramas Raros e Distintivos:** (ex: `"cfd"`, `"bar"`, `"aer"`, `"fre"`) recebem peso máximo, destacando o tópico real da pergunta.

---

## 📐 3. Similaridade Cosseno Esparsa

O cálculo de similaridade entre a pergunta $Q$ e cada âncora $A$ opera sobre mapas esparsos em memória ($O(K)$ onde $K$ é o número de trigramas da pergunta):
$$\text{Score}(Q, A) = \frac{\sum_{g \in Q \cap A} (w_Q(g) \cdot w_A(g))}{\sqrt{\sum_{g \in Q} w_Q(g)^2} \cdot \sqrt{\sum_{g \in A} w_A(g)^2}}$$

### O Portão de Conteúdo Obrigatório (*Content Token Gate*):
Mesmo que o score seja alto, a função `sharesContent()` exige que a pergunta e a âncora compartilhem **ao menos uma palavra substantiva de conteúdo com $\ge 3$ letras** (descartando stopwords e pronomes interrogativos como *qual*, *quando*, *onde*). Isso impede que perguntas aleatórias como *"qual a cor do céu?"* ativem âncoras da equipe.

---

## 📏 4. Distância de Levenshtein Dinâmica por Comprimento de Palavra

Para tolerar erros de digitação em buscas por palavras-chave sem gerar falsos positivos catastróficos, o algoritmo de Wagner-Fischer utiliza um **orçamento de edições escalonado**:

$$\text{Max Edições}(L) = \begin{cases} 0, & \text{se } L < 8\text{ letras} \\ 1, & \text{se } 8 \le L \le 11\text{ letras} \\ 2, & \text{se } L \ge 12\text{ letras} \end{cases}$$

### Evidência Empírica da Calibração:
Em palavras curtas do português, uma única edição transforma o termo em outra palavra real:
* `"materia"` $\leftrightarrow$ `"bateria"` (distância 1 $\rightarrow$ dispararia a área de baterias em uma dúvida acadêmica).
* `"falo"` $\leftrightarrow$ `"falou"` (distância 1 $\rightarrow$ dispararia despedida).
* `"quer"` $\leftrightarrow$ `"quero"` (distância 1 $\rightarrow$ dispararia afirmação).

Bloquear edições para palavras menores que 8 caracteres eliminou 100% desses falsos positivos, enquanto permitiu corrigir erros reais em palavras longas (ex: `"curriclo"` $\rightarrow$ `"curriculo"`, `"aerodinamca"` $\rightarrow$ `"aerodinamica"`).

---
* 🔗 Voltar para a [[🌐 Site UTForce - Visao Geral|Visão Geral do Site UTForce]]
* 🔗 Ver [[📝 Geracao de Linguagem Natural (NLG) Baseada em Fatos|Geração de Linguagem Natural Baseada em Fatos]]
