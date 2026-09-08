# Multiplayer Foundation

Status: **Accepted para esta fase**. Esta é a referência vigente da arquitetura da fundação.

Accepted significa decisão arquitetural aprovada para esta fase, não comportamento já implementado ou validado. Esta entrega contém documentação; ainda não há uma aplicação Unity executável. Proposed identifica uma proposta pendente de decisão; Deferred identifica trabalho adiado.

## Product Direction

Aplicativo desktop multiplayer 2D de visão lateral, com plataforma, gravidade e mundo organizado em chunks nos eixos horizontal e vertical. Minecraft e Blasphemous são referências, sem implicar terreno modificável, combate ou outros requisitos.

## First Multiplayer Milestone

Provar, em um mapa pequeno e estático, um Dedicated Server e dois clients com:

- Conexão, spawn autoritativo, controle limitado ao personagem da própria sessão e remoção no disconnect.
- Movimento horizontal, gravidade, salto e colisão calculados pelo servidor.
- Sincronização entre jogadores; prediction e reconciliation do proprietário; interpolation dos jogadores remotos.
- Travessia horizontal e vertical de chunks, incluindo salto, queda e bordas.
- AOI básico que seleciona observadores sem broadcast global de estados de movimento.
- Validação reproduzível sob latência, jitter e perda de pacotes.

Esses são objetivos de implementação e validação futura, não resultados desta Issue. Streaming visual fica depois da validação do primeiro marco.

## Technical Stack

| Responsabilidade | Decisão aceita |
| --- | --- |
| Engine e linguagem | Unity 6.3 LTS + C# |
| Servidor | Unity Dedicated Server; um único Game Server |
| Networking | FishNet |
| Transporte | Tugboat, baseado em LiteNetLib, sobre UDP |
| Estado do mundo e sessões | Memória do servidor, temporária |

O patch exato da Unity e as versões fixadas das dependências serão escolhidos, instalados e testados na Issue de inicialização. Não há versão instalada ou compatibilidade do conjunto comprovada nesta entrega. A justificativa e as alternativas estão no [ADR 001](../adr/001-engine-and-server-stack.md).

## Architecture Overview

```text
Desktop clients
  Input | Owner prediction | Rendering | Remote interpolation
                        |
                 FishNet / Tugboat
                        |
Unity Dedicated Game Server
  Sessions | Authoritative tick | Movement | Collision
  World data | Chunk index | AOI | In-memory state
```

Client e servidor usam a mesma engine e compartilham regras necessárias à simulação. A apresentação depende do estado simulado; a simulação autoritativa não depende de câmera, animação, áudio ou interface.

## Client Responsibilities

Capturar inputs; apresentar mundo e personagens; controlar câmera, animação e áudio; prever o movimento do proprietário; reconciliar seu estado com o servidor; interpolar estados dos jogadores remotos. Streaming visual será uma responsabilidade futura do client.

A colisão estática necessária à prediction local também precisa estar disponível no client. Uma entidade remota apresentada visualmente não se torna uma fonte de autoridade de gameplay.

## Server Responsibilities

Gerenciar sessões, vínculo entre sessão e personagem, spawn e disconnect; manter o estado válido do mundo; processar inputs; executar movimento, gravidade, salto e colisão; manter chunks e índice espacial; selecionar observadores por AOI; validar ações e limites de processamento.

O servidor mantém a autoridade mesmo quando o client é proprietário de um objeto de rede. Ownership identifica quem pode enviar intenção para aquele personagem.

## Shared Simulation Domain

Compartilhar convenções de coordenadas, definições de mundo, estado do controlador, contratos de dados e regras necessárias para prediction e reconciliation. Separar essas responsabilidades de componentes de apresentação.

Compartilhar código reduz duplicação, mas não torna confiáveis os dados recebidos do client nem garante determinismo entre execuções ou plataformas.

## Trust Boundaries

O client envia intenção de movimento e salto. O servidor decide posição, velocidade, contato com o chão e validade do salto.

O servidor deve validar vínculo da sessão, formato e domínio dos inputs, sequência temporal e limites de processamento. Posição, velocidade, tempo decorrido ou resultado de colisão enviados pelo client não substituem o estado autoritativo. Detalhes de payload, limites e tratamento de abuso pertencem às Issues de networking.

AOI limita replicação; não é uma permissão para executar ações nem uma garantia de sigilo. Login público e contas não fazem parte desta fase.

## Character Movement Model

Controlador explícito/cinemático: movimento horizontal, velocidade vertical, gravidade e regras de salto avançam por tick; consultas de colisão 2D limitam o deslocamento contra cenário estático.

Prediction executa essas regras somente para o jogador proprietário. Reconciliation restaura o estado autoritativo e reaplica inputs posteriores ainda pendentes, com histórico limitado. Interpolation apresenta jogadores remotos a partir de estados recebidos.

Separar posição física de suavização visual; replay não deve duplicar áudio, animações disparadas ou outros efeitos. Não prometer resultados idênticos apenas por compartilhar código. Rampas, plataformas móveis e interações físicas complexas precisam de decisões e validação próprias. Ver [ADR 002](../adr/002-authoritative-player-simulation.md).

## Coordinate Model

O mundo usa X positivo para a direita e Y positivo para cima. Uma unidade lógica equivale a um tile; pixels e escala dos sprites são apresentação. A origem matemática é (0, 0); não determina o ponto de spawn.

Posições globais são contínuas. Coordenadas de tile e de chunk são inteiras. O tile de índice (0, 0) ocupa [0, 1) em ambos os eixos; o mesmo critério de intervalos com fim exclusivo vale para outros tiles e chunks.

Para cada eixo, com tamanho N em unidades:

```text
tileCoordinate  = floor(worldPosition)
chunkCoordinate = floor(worldPosition / N)
localPosition   = worldPosition - chunkCoordinate * N
worldPosition   = chunkCoordinate * N + localPosition

0 <= localPosition < N
localTileCoordinate = floor(localPosition)
```

LocalPosition é contínua e relativa à origem do chunk; não deve ser confundida com LocalTileCoordinate. Aplicar as conversões independentemente em X e Y. Não usar resto (%) sem tratamento de negativos.

Exemplos para N = 16:

| WorldPosition | ChunkCoordinate | LocalPosition |
| --- | --- | --- |
| (0, 0) | (0, 0) | (0, 0) |
| (17.25, 31.5) | (1, 1) | (1.25, 15.5) |
| (-0.25, 16.5) | (-1, 1) | (15.75, 0.5) |
| (16, -16) | (1, -1) | (0, 0) |
| (-16.25, -0.25) | (-2, -1) | (15.75, 15.75) |

A posição exatamente em uma fronteira pertence ao intervalo que começa nessa fronteira. A fórmula inversa recompõe cada exemplo. O tamanho 16 é a baseline atual, não uma constante estrutural permanente. Limites numéricos e precisão do runtime deverão ser definidos e testados na implementação; este modelo não promete um mundo numericamente ilimitado.

## Chunk Model

| Conceito | Papel nesta fase |
| --- | --- |
| Data chunk | Agrupa dados do mapa em células de 16 × 16 tiles. |
| Visual chunk | Unidade possível de apresentação e streaming futuro. |
| Collision region | Região consultável de colisão; o pequeno mapa estático permanece residente. |
| Spatial-index chunk | Célula usada para localizar entidades candidatas a consultas e AOI. |
| Active server region | Área cuja simulação está ativa; todos os chunks do mapa inicial podem permanecer ativos. |

Esses conceitos não obrigam a usar um mesmo objeto ou ciclo de vida. Um personagem pode ocupar mais de um tile e intersectar vários chunks.

O chunk de referência localiza a entidade; não limita sua existência, colisão ou bounds. Consultas nas bordas devem considerar toda a extensão relevante e o deslocamento, inclusive em saltos e quedas. O índice deve permitir encontrar entidades que intersectam as células consultadas e evitar duplicação de uma entidade no resultado; a forma de indexar será definida na Issue correspondente.

Não há descarregamento de colisão ou transferência de autoridade ao atravessar chunks no primeiro marco.

## Interest Management

```text
Player reference chunk + 8 neighboring chunks
= initial AOI candidate region (3 × 3)
```

3 × 3 chunks é uma política inicial mensurável, não uma limitação estrutural. O servidor determina quais entidades são relevantes à conexão, considerando interseção com a região candidata e entidades nas bordas.

Ao entrar no conjunto de observadores, o client recebe a representação e o estado necessário da entidade. Ao sair, deixa de receber suas atualizações e remove essa representação. A entidade pode continuar existindo e sendo simulada no servidor. As [condições de observação do FishNet](https://fish-networking.gitbook.io/docs/guides/features/observers/custom-conditions) serão avaliadas para integrar essa política.

AOI determina entidades relevantes para replicação. Visual streaming determina conteúdo necessário para apresentação. Área visual, interesse de entidades e área de simulação são conceitos separados. Nem colisão nem simulação devem desaparecer por uma entidade sair do AOI de um jogador.

## Networking Model

```text
Input
  -> Local prediction
  -> Input command
  -> Authoritative server tick
  -> Authoritative state
       -> Owner reconciliation
       -> Remote interpolation
```

Inputs representam intenção horizontal e de salto. A implementação precisará relacionar comandos à sequência de simulação e identificar o progresso confirmado pelo servidor para reconciliar histórico pendente.

Perda, duplicação e reordenação de mensagens não podem executar o mesmo salto indevidamente nem conceder ticks extras ao client. As políticas de entrega, confirmação e recuperação serão definidas e verificadas com a versão instalada do FishNet, sem criar agora schemas finais, packet IDs ou um protocolo binário próprio.

Snapshots representam estado autoritativo em um instante. Prediction antecipa a resposta local; reconciliation corrige divergências; interpolation apresenta movimento remoto entre estados conhecidos. São responsabilidades diferentes.

## Initial Simulation Baselines

| Parâmetro | Baseline aceita e revisável |
| --- | --- |
| Dimensão do data/spatial-index chunk | 16 × 16 tiles |
| Região candidata do AOI | 3 × 3 chunks |
| Simulação autoritativa | 60 ticks/s |
| Atualização de movimento | 20 Hz |

Tick de simulação, envio de estados, entrega de inputs e frames de renderização não são a mesma frequência. A relação entre simulação e atualizações de movimento deve ser validada com o [TimeManager do FishNet](https://fish-networking.gitbook.io/docs/fishnet-building-blocks/components/managers/time-manager); estes valores não prescrevem uma configuração já testada.

Dimensões do personagem, velocidades, salto, janelas de histórico, tolerâncias de correção e buffer de interpolation ainda não estão definidos. Não deduzir capacidade de jogadores a partir destas frequências.

## Validation Direction

Preparar validação futura com servidor dedicado sem interface gráfica e dois processos de client independentes:

- Coordenadas na origem, positivos, negativos, fronteiras e conversão inversa.
- Movimento, salto, gravidade e colisão autoritativos; inputs inválidos, duplicados, atrasados e de outra sessão.
- Entrada/saída do AOI, entidades que intersectam bordas, salto/queda entre chunks e limpeza no disconnect.
- Perfis de latência de 0, 50, 100, 150 e 200 ms, registrando se o valor é RTT ou atraso por direção; combinar cenários explicitamente documentados de jitter e perda.
- Observar atraso percebido, correções, estabilidade de contato, comandos perdidos e fluidez remota; medir tempo por tick, tráfego e número de observadores.

As intensidades de jitter/perda e os limites objetivos de aceitabilidade serão definidos na Issue do ambiente de testes. Esta lista não afirma que os cenários foram executados ou que a experiência é aceitável em todas as condições.

## Deferred Systems

Fora desta fase: persistência; banco de dados; login público e contas; inventário e equipamentos; combate e habilidades; NPCs e monstros; quests e loot; crafting, comércio e economia; parties e guildas; PvP; dungeons e bosses; terreno modificável; plataformas móveis; distribuição de chunks entre servidores; microservices; infraestrutura de produção.

Esses itens não têm implementação ou arquitetura detalhada definida por esta baseline.

## Roadmap

1. Repository and Validation Foundation
2. World and Character Foundation
3. Dedicated Server and Sessions
4. Authoritative Input and Network Testbed
5. Chunk Index and Interest Management
6. Movement Synchronization and Prediction
7. First Multiplayer Milestone Validation
8. Client Visual Chunk Streaming

O GitHub é a fonte operacional do backlog. Este documento registra direção e arquitetura, sem duplicar as Issues ou autorizar a próxima automaticamente.

## Decision Revision Process

```text
Evidence -> Problem -> Alternatives -> Recommendation
-> Decision -> Documentation update
```

Uma revisão deve apresentar a evidência, o problema e os impactos das alternativas antes da decisão do responsável pelo projeto. A recomendação permanece Proposed até aprovação explícita. Atualizar esta referência e registrar ou substituir o ADR relevante quando houver mudança arquitetural significativa. Não alterar silenciosamente decisões Accepted nem transformar parâmetros experimentais em limites permanentes.

## Related ADRs

- [ADR 001 — Engine and Server Stack](../adr/001-engine-and-server-stack.md): escolhas tecnológicas e modelo de servidor.
- [ADR 002 — Authoritative Player Simulation](../adr/002-authoritative-player-simulation.md): autoridade, controlador, prediction e apresentação remota.
