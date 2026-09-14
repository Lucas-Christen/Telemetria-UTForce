---
tipo: nota-tecnica
area: faculdade
projeto: "Site UTForce"
data_atualizacao: 2026-09-13
topico: "Componente ChatWidget e Experiência de Conversa Interativa"
tags:
  - faculdade/extensao
  - react18
  - ui-ux
  - chatbot
  - componentes
---

# 💬 Componente ChatWidget & Experiência de Conversa

Especificação da interface visual do assistente interativo (`src/components/chat/ChatWidget.jsx`), máquina de estados de diálogo, sugestões dinâmicas (*chips*) e integração com formulários de candidatura.

---

## 🏎️ 1. Visão Geral do `ChatWidget`

O `<ChatWidget>` é o componente que conecta a inteligência do motor de NLP determinístico à experiência visual do visitante na landing page:

```mermaid
flowchart TD
    Botao["Botão Flutuante 'Fale Conosco'"] -->|"Clique do Usuário"| ModalChat["Janela Flutuante do Chatbot"]
    ModalChat --> Header["Header com Status e Fechar"]
    ModalChat --> MessagesList["Lista de Mensagens com Auto-Scroll"]
    ModalChat --> Chips["Sugestões Rápidas de Pergunta - Chips"]
    ModalChat --> InputBar["Barra de Digitação com Envio por Enter"]

    MessagesList --> MsgUser["Balão do Usuário - Vermelho à Direita"]
    MessagesList --> MsgBot["Balão do Bot - Carbono à Esquerda"]
    MsgBot --> Feedback["Botões de Avaliação 👍 / 👎"]
    MsgBot --> CTA["Ação Contextual: 'Abrir Inscrição'"]
```

---

## ⚙️ 2. Máquina de Estados e Contexto da Conversa

O estado do componente mantém não apenas as mensagens, mas o **contexto semântico ativo** da conversa:

```javascript
const [context, setContext] = useState({
  userName: null,       // Nome declarado pelo usuário
  lastArea: null,       // Última área de engenharia mencionada (ex: 'telemetry')
  lastIntent: null,     // Última intenção identificada
  awaitingName: false,  // Se o bot fez a pergunta "Como posso te chamar?"
  lng: 'pt-BR'          // Idioma ativo da sessão
});
```

* **Preservação de Contexto:** Se o usuário diz *"me fala da telemetria"* e logo depois pergunta *"quantas pessoas tem lá?"*, o componente injeta `lastArea: 'telemetry'` no processamento, permitindo a resolução contextual da anáfora.

---

## 🎯 3. Gatilhos de Ação Direta (*Action Triggers*)

O chatbot não é apenas informativo; ele atua como **funil de conversão**:
1. **Abertura de Formulário de Inscrição:**
   Se a pessoa perguntar sobre como entrar na equipe e confirmar interesse, a resposta do bot renderiza um botão interativo que **abre o modal de candidatura com a área já selecionada**.
2. **Sugestões Rápidas (*Quick Chips*):**
   Ao final de cada resposta, o componente oferece botões clicáveis com as dúvidas mais frequentes daquela área (ex: *"Como funciona o processo seletivo?"*, *"Quais os requisitos?"*), guiando visitantes menos propensos a digitar.
3. **Modal de Feedback com Diagnóstico:**
   Ao clicar em 👎, o usuário pode selecionar o motivo exato (*"Não entendeu o que perguntei"*, *"A resposta foi incompleta"* ou *"Não era bem isso"*), alimentando o índice de curadoria do Supabase.

---
* 🔗 Voltar para a [[🌐 Site UTForce - Visao Geral|Visão Geral do Site UTForce]]
* 🔗 Ver [[🪜 Cascata de Decisao de Intencoes em 8 Camadas|Cascata de Decisão de Intenções]]
