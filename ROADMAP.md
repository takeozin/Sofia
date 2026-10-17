# Roadmap: Sofia (Bot de Estudos para Telegram)

> **Status (17/10/2026): fase 1 concluída.** A Sofia roda no n8n, responde texto e imagens no Telegram com respostas objetivas e só atende chats liberados. Ainda **não conhece quem está do outro lado**: cada conversa começa do zero, sem saber o curso, o cargo ou o objetivo do aluno, e o histórico some se o n8n reiniciar.

Autor: Lucas Takeo. Repositório: `takeozin/Sofia`.

## 📊 Baseline (17/10/2026)
- Workflow com 15 nós em `sofia_bot_github.json` (print em `docs/workflow-n8n.png`)
- Fluxo: Telegram Trigger → Chat Autorizado? → Foto ou Texto? → Tutor de Estudos → Formatar Resposta → Enviar Resposta
- Acesso liberado por lista fixa de chat_ids escrita no próprio nó **Chat Autorizado?** (`COLOQUE_SEUS_CHAT_IDS_AQUI`)
- Memória: **Simple Memory** com as últimas 10 mensagens por chat, **guardada na RAM** do n8n (some ao reiniciar)
- Nenhum dado sobre o aluno: o prompt é igual para todo mundo
- LLM genérica: o nó **LLM de Sua Preferência** vem com o Google Gemini, mas aceita qualquer Chat Model do n8n
- Respostas divididas em partes de até 3500 caracteres e enviadas em HTML, com reenvio em texto puro se o Telegram recusar
- Formatos suportados: texto, foto e arquivo de imagem. Áudio, vídeo, sticker e PDF recebem aviso de "não suportado"
- O comando `/start` do Telegram vai direto para a LLM (não há boas-vindas fixas)

## 🧠 Fase 2: banco de dados e perfil do aluno (fazer agora)

Objetivo: a Sofia lembra quem é cada pessoa (nome, curso, período, cargo atual, objetivo) e adapta as respostas a isso. Também passa a ter memória de conversa que não se perde ao reiniciar o n8n.

### 1. Criar o banco · importância alta · dificuldade média
É a base de toda a fase. Sem ele nenhum dos itens seguintes funciona.
- [ ] Decidir onde fica o banco (ver **Decisões pendentes**)
- [ ] Criar a tabela `perfis`, com uma linha por aluno:
  - `chat_id` (chave primária, o ID do chat no Telegram)
  - `nome`
  - `curso`, `instituicao`, `periodo`
  - `cargo_atual` (ex.: estagiário de suporte, desenvolvedor júnior, não trabalha na área)
  - `objetivo` (ex.: passar em concurso, TCC, mudar de área)
  - `nivel` (iniciante, intermediário, avançado)
  - `observacoes` (preferências livres: "prefere exemplos em Python", "tem dificuldade em cálculo")
  - `criado_em`, `atualizado_em`
- [ ] Salvar a criação da tabela como migration no repositório (`supabase/migrations/`), para quem clonar recriar o banco
- [ ] Ativar RLS sem nenhuma policy pública: só o n8n acessa, por conexão direta ao Postgres
- [ ] Credencial do Postgres no n8n e uma linha no README explicando como configurar

### 2. Memória de conversa persistente · importância alta · dificuldade baixa
Aproveita o banco do item 1 e resolve a limitação conhecida do README.
- [ ] Trocar o **Simple Memory** pelo **Postgres Chat Memory**, mantendo `chat_id` como chave da sessão
- [ ] Manter a janela de 10 mensagens (mais que isso aumenta o custo de cada chamada à LLM)
- [ ] Testar: conversar, reiniciar o n8n e conferir se a Sofia lembra da conversa
- [ ] Remover a limitação "memória fica na RAM" do README

### 3. Carregar o perfil antes de responder · importância alta · dificuldade média
É aqui que o perfil passa a mudar as respostas.
- [ ] Novo nó **Buscar Perfil** (Postgres, `SELECT` por `chat_id`) logo depois de **Chat Autorizado?**
- [ ] Incluir o perfil no System Message do **Tutor de Estudos** (ex.: "Você está falando com Ana, 5º período de Engenharia de Software, estagiária de QA, quer passar em concurso")
- [ ] Instruir a Sofia a usar o perfil para escolher exemplos e profundidade, sem ficar repetindo os dados para o aluno
- [ ] Tratar aluno sem perfil: responder normalmente e seguir para o item 4

### 4. Cadastro e atualização do perfil · importância alta · dificuldade média
- [ ] `/start` com boas-vindas fixas (sem gastar a cota da LLM) e as perguntas iniciais: nome, curso, período, cargo atual e objetivo
- [ ] Ferramenta (*tool*) **Atualizar Perfil** ligada ao **Tutor de Estudos**, para a Sofia salvar sozinha o que o aluno contar no meio da conversa ("passei pro 6º período", "fui efetivado")
- [ ] Comando `/perfil` para o aluno ver o que a Sofia sabe sobre ele
- [ ] Testar: cadastrar, mudar o cargo pela conversa e conferir no banco

### 5. Privacidade (LGPD) · importância alta · dificuldade baixa
O banco passa a guardar dados pessoais, então o aluno precisa saber e poder apagar.
- [ ] Avisar no `/start` quais dados são guardados e para quê
- [ ] Comando `/esquecer` que apaga o perfil e o histórico de conversa daquele `chat_id`
- [ ] Não guardar o conteúdo das imagens enviadas, só o texto da conversa
- [ ] Seção "Privacidade" no README

## 🚀 Fase 3: novos formatos e acesso (fazer depois)

Objetivo: a Sofia entende mais tipos de mensagem e fica mais fácil de liberar para novas pessoas. Fica separada porque não depende do banco e não bloqueia a fase 2.

### 6. Mensagens de voz · importância média · dificuldade média
- [ ] Transcrever o áudio (ex.: Whisper ou o próprio modelo, se for multimodal com áudio) e mandar o texto ao Tutor
- [ ] Trocar o aviso de "não suportado" para áudio

### 7. PDF · importância média · dificuldade média
- [ ] Extrair o texto do PDF (nó **Extract From File**) e mandar ao Tutor
- [ ] Limitar o tamanho para não estourar o contexto da LLM

### 8. Liberação de acesso pelo banco · importância baixa · dificuldade baixa
Só faz sentido com o banco da fase 2.
- [ ] Coluna `autorizado` em `perfis`, substituindo a lista fixa no nó **Chat Autorizado?**
- [ ] Liberar uma pessoa nova sem precisar editar o workflow

## ❓ Decisões pendentes
- [ ] Onde fica o banco: **Supabase** (Postgres gerenciado, já usado no formulário interativo, plano gratuito) ou Postgres na mesma máquina do n8n? Recomendação: Supabase, num **projeto novo**, separado do formulário
- [ ] O cadastro no `/start` é obrigatório antes de usar, ou a Sofia responde de qualquer jeito e completa o perfil aos poucos?
- [ ] Quais campos são obrigatórios? (sugestão: só `nome` e `curso`; o resto é opcional)

## 🔄 Sequência de Próximos Passos

1. **Fechar as decisões pendentes** (onde fica o banco e como funciona o cadastro).
2. **Criar o banco e a memória persistente** (itens 1 e 2). Testar reiniciando o n8n.
3. **Buscar o perfil e incluir no prompt** (item 3), com um perfil cadastrado direto no banco para testar.
4. **Cadastro pelo Telegram e privacidade** (itens 4 e 5) e atualização do README com um print do perfil funcionando.
5. **Começar a fase 3** pela voz (item 6), que é o formato mais usado no celular.

## ✅ Concluído (fase 1)
* **Bot no Telegram com n8n:** texto e imagens (foto ou arquivo), usando a imagem na maior resolução.
* **LLM de sua preferência:** o nó de modelo é genérico e o README explica como trocar de provedor. Use um modelo multimodal para entender imagens.
* **Respostas objetivas:** resposta primeiro, explicação em até 3 parágrafos e pegadinhas comuns. Explicação longa só quando o aluno pede.
* **Modo `/dica`:** só pistas, sem entregar a resposta.
* **Respostas longas:** divididas em várias mensagens, sem cortar blocos de código no meio.
* **Formatação segura:** Markdown convertido para o HTML do Telegram, com reenvio em texto puro se falhar.
* **Acesso restrito:** só responde a chat_ids liberados; quem não está liberado recebe o próprio chat_id.
* **Formatos não suportados:** aviso ao aluno, em vez de mandar uma mensagem vazia para a LLM.
* **Repositório:** README com prints, diagrama do workflow e limitações; licença MIT; tags e descrição no GitHub.
