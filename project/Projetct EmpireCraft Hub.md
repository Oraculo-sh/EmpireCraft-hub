# **Projeto: EmpireCraft Hub (Fundação) \- Documento de Design**

Versão: 5.0 (UI Recolhível e Estado Privado do Jogo)  
Data: 25 de outubro de 2025  
Proprietário do Projeto: (Nome do Gerente de Projeto/Líder da Equipe)

## **1\. Introdução**

### **1.1. Visão do Produto**

Criar a plataforma "EmpireCraft Hub" como uma Atividade centralizada do Discord. Esta fundação servirá como um hub social e comercial, gerenciando perfis de usuário, estatísticas, customização de avatar e uma loja de cosméticos. A plataforma será projetada de forma modular para permitir a integração futura de diversos jogos de tabuleiro e cartas.

A experiência é focada em sessões sociais: a Atividade é instanciada por canal de voz. Todos os usuários no mesmo canal de voz compartilham um lobby, e o jogo acontece em tempo real para os participantes daquele lobby.

### **1.2. Escopo (Fases 1-4)**

Este documento detalha o desenvolvimento da **Plataforma Fundação**. Os jogos em si são considerados "módulos" a serem desenvolvidos *após* a conclusão bem-sucedida deste escopo.

**O que ESTÁ no escopo (In-Scope):**

* **Arquitetura de Lobby por Canal de Voz:** O sistema deve tratar cada canal de voz do Discord como uma instância de lobby separada e única.  
* **Líder de Sala (Host):** O primeiro usuário a iniciar a Atividade em um canal de voz é designado como o "Líder" (Host), com permissões para controlar a Tela Principal.  
* **UI de Tela Dupla (Recolhível):**  
  * Uma **Tela Principal** (sincronizada para todos no lobby) que exibe o menu de seleção de jogos ou a partida em andamento.  
  * Uma **Tela Secundária** (individual) que exibe os menus de usuário (Perfil, Loja, etc.) e um Mini-Chat.  
  * Um **Botão de Controle da UI** para recolher/expandir a Tela Secundária, permitindo que a Tela Principal ocupe a janela inteira.  
* **Arquitetura de Jogo (Estado Público/Privado):** A arquitetura do backend deve ser projetada para enviar dois tipos de estado de jogo:  
  * **Estado Público** (ex: o tabuleiro), transmitido para todos no lobby.  
  * **Estado Privado** (ex: suas cartas), enviado individualmente para cada jogador.  
* **Sistema de Participantes (Join/Espectador):** Usuários no lobby podem optar por "Entrar" (player) na partida ou permanecer como spectator.  
* **Mini-Chat de Lobby:** Um chat embutido na Tela Secundária, exclusivo para o lobby, com suporte a emojis básicos e "premium" (itens cosméticos).  
* **Autenticação e Persistência:** Sistema completo de conta de usuário (moedas, XP, inventário).  
* **Sistemas de Monetização e Economia:** Fluxos completos de compra (Dinheiro Real \-\> Coins) e (Coins \-\> Cosméticos).  
* **Perfis e Customização:** Telas de perfil, leaderboards e customização de avatar.  
* **Catálogo de Jogos (Mockado):** A interface na Tela Principal para o Líder selecionar um jogo (com selo "Em Breve").

**O que NÃO ESTÁ no escopo (Out-of-Scope):**

* A lógica de *qualquer* jogo (War, Detetive, Monopoly, etc.).  
* A implementação da lógica de jogo que permite "entrar durante a partida" (isso será definido por cada módulo de jogo).  
* Jogabilidade entre diferentes canais de voz (cross-lobby).

## **2\. Requerimentos**

(Esta seção permanece em grande parte a mesma da v4.0, pois a stack de tecnologia é idêntica)

### **2.1. Ambiente de Desenvolvimento**

* **Runtime:** Node.js (LTS recomendado).  
* **Controle de Versão:** Git (com fluxo de trabalho Git Flow: main, develop, feature/\*).  
* **Ambiente Local:** Docker Compose para padronizar os serviços de banco de dados (postgres, redis) localmente.

### **2.2. Linguagens**

* **Linguagem Principal:** TypeScript (para Frontend e Backend).  
* **Banco de Dados:** SQL (PostgreSQL) e Prisma Schema Language.

### **2.3. Dependências Chave (Stack de Tecnologia)**

| Camada | Tecnologia (Framework/Dependência) | Propósito |
| :---- | :---- | :---- |
| **Frontend** | React | Biblioteca principal da UI. |
|  | Vite | Build tool para desenvolvimento e build. |
|  | React Router | Para navegação SPA *dentro da Tela Secundária*. |
|  | Zustand | Gerenciamento de estado global (usuário, lobby, jogo). |
|  | socket.io-client | Cliente WebSocket. |
|  | axios (ou fetch) | Cliente HTTP para API REST. |
|  | Discord Embedded App SDK | Ponte de autenticação, monetização e *contexto de lobby*. |
| **Backend** | NestJS | Framework principal (API REST e WebSockets). |
|  | Socket.io | Biblioteca do servidor WebSocket (com Rooms). |
|  | Prisma | ORM para PostgreSQL. |
| **Database** | PostgreSQL | Banco de dados relacional principal (usuários, inventário, etc.). |
|  | Redis | Cache (leaderboards) e **Armazenamento de Estado de Lobby Ativo**. |

## **3\. Desenvolvimento (Arquitetura e Design)**

Esta seção foi revisada para refletir a nova arquitetura de UI e Lobby.

### **3.1. Arquitetura Detalhada e Componentes**

1. **Frontend (React SPA \- Layout de Duas Colunas):**  
   * **Função:** Camada de apresentação. A raiz do app React renderiza um layout dividido: \[MainView\] e \[SidebarView\].  
   * **Estado da UI Local (Zustand):** O frontend manterá um estado local, isSidebarCollapsed: boolean. Um botão de controle alternará esse estado, e o CSS (via classes) ajustará o layout (ex: flex-grow: 0 para a Sidebar, width: 100% para o MainView). Este estado *não* é sincronizado.  
   * **\[MainView\] (Tela Principal):**  
     * Renderiza o estado *público* do jogo (ex: lobby.publicGameState.board) que é recebido por todos.  
     * Renderiza condicionalmente o estado *privado* do jogo (ex: user.privateGameState.hand) que é recebido apenas por você.  
     * **Exemplo:** Todos veem o tabuleiro. Apenas você vê suas cartas renderizadas na parte inferior da \[MainView\].  
     * **Controle:** Apenas o host pode disparar ações de lobby (ex: emit('select-game')). Durante um jogo, apenas o jogador da vez pode disparar ações de jogo.  
   * **\[SidebarView\] (Tela Secundária):**  
     * Este componente é **individual**.  
     * Usa React Router *internamente* para navegar entre /profile, /store, /settings.  
     * Contém o componente MiniChat na parte inferior, que é persistente.  
   * **DiscordSDKService:** (Papel crucial)  
     * Na inicialização, deve obter o discord\_id (para auth) e o **voice\_channel\_id** (para o lobby).  
2. **Backend (NestJS API Autoritativa \+ Gerenciador de Lobby):**  
   * **Função:** Cérebro, árbitro e fonte da verdade.  
   * **API REST:** Como antes (dados frios: loja, catálogo de jogos).  
   * **API WebSocket (Gateway):** Gerencia "Lobbys".  
     * Quando um usuário se conecta com emit('auth'), o backend usa o voice\_channel\_id para encontrar ou criar um **Estado de Lobby** no **Redis**.  
     * O backend coloca o socket do usuário em uma **Sala (Room) do Socket.io** nomeada com o voice\_channel\_id.  
   * **Gerenciador de Lobby (Redis):**  
     * Cada canal de voz com uma atividade ativa terá um objeto no Redis. Ex: lobby:{voice\_channel\_id}.  
     * Este objeto armazena: host\_id, current\_screen, participants: \[...\], chat\_history: \[\], e o **public\_game\_state** (o tabuleiro, de quem é a vez, etc.).  
   * **Gerenciador de Jogo (Memória do Backend \- Fase 5 Futura):**  
     * Quando um jogo iniciar, o backend irá instanciar uma classe (ex: new DetectiveGame(...)).  
     * Essa classe manterá o estado *privado* (as cartas de cada um) em sua memória interna. Ela será responsável por enviar atualizações privadas diretamente para os sockets dos jogadores.  
3. **Database (PostgreSQL \+ Redis):**  
   * **PostgreSQL:** O "cofre" (como antes). Armazena dados persistentes (usuários, inventário, etc.).  
   * **Redis:** (Papel expandido) Armazena dados voláteis (cache de leaderboard) e todo o estado de sessão dos **Lobbys Ativos** (incluindo o public\_game\_state).

### **3.2. Fluxo de Dados: REST vs. WebSockets**

(Permanece o mesmo da v4.0)

### **3.3. Modelo de Dados (Prisma Schema \- v1.0)**

(Nenhuma mudança necessária no schema.prisma da v4.0. O estado do jogo (público/privado) é volátil e vive no Redis ou na memória do servidor, não no PostgreSQL).

## **4\. Funções (Features e API)**

Esta seção foi revisada para detalhar a nova API de Lobby e a arquitetura de estado público/privado.

### **4.1. Contrato de API: Endpoints REST (Controladores NestJS)**

(Nenhuma mudança da v4.0. GET /api/games, GET /api/categories, GET /api/store/items, GET /api/leaderboard, POST /webhook/discord-payment permanecem os mesmos.)

### **4.2. Contrato de API: Eventos WebSocket (Gateways NestJS)**

* **Gateway Principal:** LobbyGateway  
* **(Eventos de Lobby e Chat \- da v4.0, sem alterações):**  
  * auth (Cliente \-\> Servidor)  
  * join-game (Cliente \-\> Servidor)  
  * select-game (Cliente \-\> Servidor, Somente Host)  
  * start-game (Cliente \-\> Servidor, Somente Host)  
  * send-chat-message (Cliente \-\> Servidor)  
* **(Eventos de Economia \- da v4.0, sem alterações):**  
  * buy-item (Cliente \-\> Servidor)  
  * equip-item (Cliente \-\> Servidor)  
  * unequip-item (Cliente \-\> Servidor)  
* **Eventos de Estado de Jogo (Design para Fase 5 \- Módulos de Jogo):**  
  * A arquitetura do LobbyGateway deve ser capaz de lidar com os seguintes eventos quando os módulos de jogo forem implementados.  
* **Evento (Servidor \-\> Cliente, Broadcast de Sala):** on('game-state-public')  
  * **Payload:** { public\_state: any } (ex: { board: \[...\], current\_turn\_id: '...' })  
  * **Descrição:** Transmite o estado público do jogo para **todos** na sala (jogadores e espectadores). Atualiza o public\_game\_state no Redis.  
  * **Ação do Frontend:** O useLobbyStore (Zustand) recebe isso e atualiza seu estado, fazendo com que a \[MainView\] renderize o tabuleiro atualizado.  
* **Evento (Servidor \-\> Cliente, Envio Direto):** on('game-state-private')  
  * **Payload:** { private\_state: any } (ex: { hand: \['card1', 'card2'\], secret\_role: 'detective' })  
  * **Descrição:** Enviado *apenas* para o socket do jogador relevante. **Não** é transmitido para a sala.  
  * **Ação do Frontend:** O useUserStore (Zustand) recebe isso e atualiza o estado privado do jogador. A \[MainView\] (renderizando no cliente desse usuário) detecta essa mudança e renderiza a "mão" de cartas do jogador.  
* **Evento (Cliente \-\> Servidor):** game-action  
  * **Payload:** { action: string, payload: any } (ex: { action: 'play\_card', payload: { card\_id: '...' } })  
  * **Descrição:** O jogador (da vez) envia uma ação de jogo.  
  * **Ação (Backend):** O LobbyGateway encaminha esta ação para a instância de jogo correta (ex: DetectiveGame.handleAction(...)). A instância processa a lógica, atualiza o estado e, em seguida, dispara os eventos on('game-state-public') (broadcast) e/ou on('game-state-private') (direto) necessários.

### **4.3. Fluxo de Usuário (Exemplo: Jogo com Dados Sigilosos)**

(Os passos 1-22 do Fluxo de Usuário da v4.0 permanecem idênticos. O fluxo abaixo começa após o Host iniciar o jogo)

...  
22\. Frontend (A e B): Sua \[MainView\] muda para a tela de lobby do jogo Detetive.  
...  
23\. Usuário B: Clica em "Entrar na Partida".  
24\. Frontend (B): Emite emit('join-game').  
25\. Backend: Atualiza Redis (UserB é player). Emite on('lobby-state-update').  
26\. Usuário A (Host): Clica em "Iniciar Partida".  
27\. Frontend (A): Emite emit('start-game').  
28\. Backend: (Verifica se A é host). Inicia a Fase 5 (Futuro):  
\* Cria uma instância: const game \= new DetectiveGame(\['UserA', 'UserB'\]).  
\* A instância game gera o estado.  
\* game.getPublicState() retorna { board: {...}, turn: 'UserA' }.  
\* game.getPrivateState('UserA') retorna { hand: \['cardA', 'cardB'\], role: 'X' }.  
\* game.getPrivateState('UserB') retorna { hand: \['cardC', 'cardD'\], role: 'Y' }.  
29\. Backend: Emite on('main-screen-update', { screen: 'detective\_game\_board' }) para toda a sala VC1.  
30\. Backend: Emite on('game-state-public', { board: {...}, turn: 'UserA' }) para toda a sala VC1.  
31\. Backend: Emite on('game-state-private', { hand: \['cardA', 'cardB'\], role: 'X' }) apenas para o socket do Usuário A.  
32\. Backend: Emite on('game-state-private', { hand: \['cardC', 'cardD'\], role: 'Y' }) apenas para o socket do Usuário B.  
33\. Frontend (A e B):  
\* Ambos useLobbyStore (Zustand) recebem o game-state-public.  
\* Sua \[MainView\] renderiza o DetectiveGameBoard.  
\* Ambos veem o tabuleiro e que é a vez do UserA.  
34\. Frontend (A):  
\* useUserStore (Zustand) recebe o game-state-private.  
\* A \[MainView\] renderiza a "mão" de cartas (CardA, CardB) na parte inferior da tela, visível apenas para A.  
35\. Frontend (B):  
\* useUserStore (Zustand) recebe seu próprio game-state-private.  
\* A \[MainView\] renderiza a "mão" de cartas (CardC, CardD) na parte inferior da tela, visível apenas para B.

## **5\. Roadmap (Etapas de Desenvolvimento)**

(Revisado para incluir a lógica de UI Recolhível)

### **Fase 1: Fundação, Autenticação e Lógica de Lobby**

*O objetivo é ter o lobby multiusuário funcional, com sync de participantes.*

* **Data (DBA / Equipe Back):**  
  * \[ \] Definir schema User.  
  * \[ \] Configurar Docker Compose (Postgres, Redis).  
  * \[ \] Rodar migração.  
* **Backend (Equipe Back):**  
  * \[ \] Setup NestJS \+ Prisma \+ Socket.io \+ Redis.  
  * \[ \] Criar o LobbyGateway.  
  * \[ \] Implementar o evento socket.on('auth') (com lógica "Get or Create User" no Postgres e "Get or Create Lobby" no Redis).  
  * \[ \] Implementar o socket.join(voice\_channel\_id).  
  * \[ \] Implementar on('auth-success') (resposta individual) e on('lobby-state-update') (resposta para a sala).  
  * \[ \] Implementar socket.on('join-game') (muda status no Redis e dispara lobby-state-update).  
* **Frontend (Equipe Front):**  
  * \[ \] Setup React \+ Vite \+ Zustand \+ Socket.io-client.  
  * \[ \] Implementar DiscordSDKService (para authenticate E getChannel()).  
  * \[ \] Implementar lógica de inicialização (Auth \+ Conexão com Lobby).  
  * \[ \] Criar useUserStore e useLobbyStore (Zustand).  
  * \[ \] Ouvir por auth-success e lobby-state-update e salvar no Zustand.  
  * \[ \] Criar layout de duas colunas: \[MainView\] e \[SidebarView\].  
  * \[ \] \[SidebarView\] deve ter um componente ParticipantList que lê do useLobbyStore.  
  * \[ \] \[SidebarView\] deve ter um botão "Entrar" que emite emit('join-game').

### **Fase 2: Navegação, Telas Principais (Sincronizadas) e UI Recolhível**

*O objetivo é construir a navegação da Tela Secundária, a seleção de jogos sincronizada na Tela Principal e o controle da UI.*

* **Data (DBA / Equipe Back):**  
  * \[ \] Adicionar schemas GameCatalog e GameCategory.  
  * \[ \] Criar *script seed* para popular os jogos (marcados is\_available \= false).  
* **Backend (Equipe Back):**  
  * \[ \] Implementar GET /api/games e GET /api/categories.  
  * \[ \] Implementar socket.on('select-game') (somente host).  
  * \[ \] Implementar on('main-screen-update') (push para a sala).  
* **Frontend (Equipe Front):**  
  * \[ \] Na \[SidebarView\], implementar React Router com as Rotas (/profile, /store, /settings) e Páginas correspondentes.  
  * \[ \] Criar NavBar na \[SidebarView\].  
  * \[ \] **\[MainView\]:**  
    * \[ \] Deve ler o current\_screen do useLobbyStore.  
    * \[ \] Criar GameSelectionScreen: Chama GET /api/games e renderiza os cards.  
    * \[ \] Implementar onClick nos cards que emite emit('select-game') (habilitado *apenas* se o user.id \=== lobby.host\_id).  
  * \[ \] **Layout Global:**  
    * \[ \] Adicionar um estado ao useLobbyStore (ou um store local) isSidebarCollapsed: boolean.  
    * \[ \] Adicionar um botão (ícone de seta) que alterna esse estado.  
    * \[ \] Ajustar o CSS/Layout para que \[SidebarView\] recolha e \[MainView\] expanda.  
  * \[ \] **Página Settings (Rota /settings na Sidebar):**  
    * \[ \] Criar a UI de configurações (lógica no localStorage).

### **Fase 3: Economia, Loja e Mini-Chat**

*O objetivo é ter o fluxo completo de compra e o mini-chat funcional.*

* **Data (DBA / Equipe Back):**  
  * \[ \] Adicionar schemas StoreItem, UserInventory, ItemType.  
  * \[ \] Criar *script seed* para cosméticos (incluindo "pacotes de emoji premium").  
* **Backend (Equipe Back):**  
  * \[ \] **Monetização Discord:**  
    * \[ \] Configurar SKUs (pacotes de "coins").  
    * \[ \] Implementar POST /webhook/discord-payment (com validação e emit('balance-updated')).  
  * \[ \] **Loja Interna:**  
    * \[ \] Implementar GET /api/store/items.  
    * \[ \] Implementar socket.on('buy-item') (lógica transacional).  
  * \[ \] **Mini-Chat:**  
    * \[ \] Implementar socket.on('send-chat-message') (sanitizar, checar emojis premium, salvar no chat\_history do Redis).  
    * \[ \] Implementar on('new-chat-message') (push para a sala).  
* **Frontend (Equipe Front):**  
  * \[ \] **Página Store (Rota /store na Sidebar):**  
    * \[ \] Implementar UI da loja (chamar API) e botões de compra (chamar SDK e emit('buy-item')).  
  * \[ \] **Zustand (useUserStore):**  
    * \[ \] Implementar ouvintes para balance-updated e buy-success.  
  * \[ \] **\[SidebarView\]:**  
    * \[ \] Criar componente MiniChat.  
    * \[ \] Deve ler o chat\_history do useLobbyStore.  
    * \[ \] Deve ouvir por on('new-chat-message') e adicionar à lista.  
    * \[ \] Deve ter um input que emite emit('send-chat-message').

### **Fase 4: Perfil, Stats e Customização**

*O objetivo é permitir que o usuário se personalize e veja seu progresso.*

* **Data (DBA / Equipe Back):**  
  * \[ \] Criar índices no DB para User(xp), User(wins).  
* **Backend (Equipe Back):**  
  * \[ \] Implementar GET /api/leaderboard?stat=... (com cache Redis).  
  * \[ \] Implementar socket.on('equip-item') e socket.on('unequip-item').  
  * \[ \] Implementar on('avatar-updated').  
* **Frontend (Equipe Front):**  
  * \[ \] **Página Profile (Rota /profile na Sidebar):**  
    * \[ \] Criar abas (Stats, Avatar, Leaderboard).  
    * \[ \] Aba "Stats": Ler dados do useUserStore.  
    * \[ \] Aba "Leaderboard": Chamar GET /api/leaderboard e renderizar.  
  * \[ \] **Página AvatarCustomization (Aba /profile/avatar):**  
    * \[ \] Ler inventory do useUserStore.  
    * \[ \] Renderizar AvatarPreview e lista de itens.  
    * \[ \] Implementar onClick para "Equipar" (emit('equip-item')).  
  * \[ \] **Layout (Global):**  
    * \[ \] Conectar o avatar do usuário (no Header ou Perfil) e o MiniChat (para emojis) ao useUserStore para que reflitam as mudanças de inventário e avatar-updated.