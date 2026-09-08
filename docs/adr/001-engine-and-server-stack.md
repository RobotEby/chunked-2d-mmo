# ADR 001 — Engine and Server Stack

Status: **Accepted para esta fase**

Decision date: 2026-09-08

Accepted registra aprovação arquitetural; não comprova instalação, integração ou validação prática. A [baseline central](../architecture/multiplayer-foundation.md) é a referência vigente.

## Context

Precisamos iniciar um jogo desktop 2D lateral multiplayer com controle responsivo, gravidade, colisão e servidor dedicado. O primeiro marco envolve dois clients, um mapa estático e uma única autoridade de mundo.

Manter regras e consultas de colisão consistentes entre o servidor e a prediction local é mais urgente que reduzir ao mínimo o custo de cada processo ou distribuir a simulação.

## Decision

Usar Unity 6.3 LTS e C# para client e Unity Dedicated Server, com FishNet para networking e Tugboat, baseado em LiteNetLib/UDP, como transporte inicial. Manter um único Game Server nesta fundação.

A [Unity 6.3 LTS](https://unity.com/blog/unity-6-3-lts-is-now-available) define a linha de engine aprovada. O [build target Dedicated Server](https://docs.unity3d.com/6000.3/Documentation/Manual/dedicated-server-introduction.html) é a base escolhida para a aplicação de servidor. O [Tugboat](https://fish-networking.gitbook.io/docs/fishnet-building-blocks/transports/tugboat) utiliza [LiteNetLib](https://github.com/RevenantX/LiteNetLib), biblioteca sobre UDP.

O patch exato da engine e as versões de FishNet e dependências serão fixados durante a inicialização, após instalação e testes. Links de documentação não especificam dependências já instaladas.

## Alternatives considered

| Alternativa | Motivo para não escolher nesta fase |
| --- | --- |
| Unreal Engine | Exigiria outra abordagem de engine e workflow; não identificamos um ganho para este primeiro marco 2D que justifique preferi-la à baseline escolhida. |
| Godot | Continua viável. A decisão favorece o caminho Unity/C# e FishNet para manter uma única linha de investigação de servidor e prediction, em vez de validar duas stacks. |
| Servidor independente em C# ou outra linguagem | Facilitaria reduzir a dependência do runtime da engine, mas exigiria reproduzir ou substituir consultas de colisão e manter compatibilidade com o client. Esse custo não se justifica antes do primeiro marco. |
| Netcode for GameObjects | A documentação de [client anticipation](https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.13/manual/advanced-topics/client-anticipation.html), consultada em 2026-09-08, distingue anticipation de um ciclo completo de prediction com rollback/replay. Isso deixaria mais trabalho próprio para a necessidade escolhida. |
| Mirror | É uma alternativa a considerar se a integração escolhida falhar. Preferimos começar com o fluxo documentado de replicate/reconcile do FishNet; não houve benchmark comparativo nem prova de inadequação do Mirror. |

FishNet foi escolhido pelo encaixe do [fluxo de prediction e reconciliation](https://fish-networking.gitbook.io/docs/guides/features/prediction/creating-code/controlling-an-object) no modelo aprovado. Isso não elimina o trabalho de construir e validar nosso controlador 2D.

## Consequences

- Compartilhar engine, linguagem, definições e regras reduz a necessidade de manter duas implementações de simulação.
- O build dedicado permite separar a execução do servidor da apresentação; essa separação precisa existir também nas dependências do código.
- O servidor depende do runtime, das ferramentas e do ciclo de atualização da Unity. Seu consumo de recursos e processo de build ainda precisam ser medidos.
- FishNet e Tugboat criam dependências de APIs, comportamento e manutenção externos. Atualizações precisam de validação e versões reproduzíveis.
- Compartilhar engine não garante determinismo, ausência de divergências ou compatibilidade automática entre todos os componentes.
- UDP exige tratar perda, reordenação e duplicação no fluxo de comandos/estados; não há promessa de robustez apenas pela escolha do transporte.
- Servidor independente permanece uma alternativa de revisão, sem criar uma camada genérica ou serviço adicional antecipadamente.

Mudanças seguem o processo de revisão da baseline central, com evidência e aprovação explícita.
