# ADR 002 — Authoritative Player Simulation

Status: **Accepted para esta fase**

Decision date: 2026-09-08

Accepted registra aprovação arquitetural; não significa comportamento implementado ou validado. A [baseline central](../architecture/multiplayer-foundation.md) é a referência vigente.

## Context

Um jogo de plataforma multiplayer precisa responder rapidamente ao input sem permitir que o client decida posição válida, velocidade ou resultado de um salto. Latência e divergências entre execuções tornam insuficiente compartilhar apenas a posição atual.

O primeiro marco permite restringir a colisão a um cenário estático residente, para investigar movimento e sincronização antes de interações físicas complexas.

## Decision

- O servidor mantém o estado autoritativo e valida inputs da sessão responsável pelo personagem.
- O client envia intenção; posição, velocidade e contato com o chão são resultados da simulação.
- O controlador usa movimentação explícita/cinemática e consultas de colisão 2D, com movimento horizontal, gravidade e regras de salto atualizados por tick.
- Somente o proprietário executa prediction do próprio personagem.
- Reconciliation restaura um estado confirmado e reaplica inputs pendentes posteriores ao estado, corrigindo divergências.
- Jogadores remotos usam interpolation de estados recebidos.
- O pequeno cenário de colisão estática permanece residente no primeiro marco.

A [documentação de prediction do FishNet](https://fish-networking.gitbook.io/docs/guides/features/prediction/creating-code/controlling-an-object) orientará a integração. Seus exemplos não são um controlador de plataforma pronto nem evidência de que nosso motor já funciona.

## Alternatives considered

| Alternativa | Benefício e custo relevante |
| --- | --- |
| Client-authoritative | Facilita resposta local, mas permite que um client adulterado declare movimento impossível como válido. Não atende à fronteira de confiança aprovada. |
| Rigidbody2D baseado prioritariamente em forças | Pode facilitar interações físicas emergentes, mas introduz estados e dependências do solver que precisam ser reproduzidos durante correções. Para este marco estático, preferimos explicitar as regras do controlador. |
| Prediction de todas as entidades em todos os clients | Pode atender interações previstas mais complexas, mas amplia histórico, dependências e trabalho de replay. Não é necessária para observar outros jogadores neste marco. |

Movimentação explícita não proíbe um uso auxiliar de componentes físicos. O detalhe de componentes e APIs será decidido na implementação, preservando a responsabilidade do motor pelas regras de movimento.

## Consequences

- O estado necessário para reproduzir o controlador precisa ser explícito e restaurável, incluindo variáveis que afetem o próximo tick; posição isolada pode ser insuficiente.
- O histórico de inputs e estados deve ser limitado. Capacidade, descarte e recuperação quando o histórico não basta precisam de validação na Issue correspondente.
- Reexecutar um input para reconstruir estado não deve duplicar efeitos como áudio ou eventos de apresentação.
- A posição usada por colisão e simulação deve ser separada da suavização visual. Suavizar a representação não pode alterar silenciosamente o resultado autoritativo.
- Regras, definições do cenário e consultas precisam permanecer consistentes entre proprietário e servidor. Mesmo assim, divergências devem ser esperadas e medidas.
- Os clients remotos não se tornam autoridades físicas por exibirem personagens interpolados.
- Rampas, plataformas móveis e interações físicas complexas exigirão avaliação própria; a decisão atual não comprova suporte a elas.
- Jitter, perda e latência podem expor correções perceptíveis. A qualidade do movimento será validada com o ambiente de testes, sem prometer antecipadamente um limite aceitável.

As frequências e os parâmetros experimentais ficam na baseline central, permitindo revisá-los sem duplicar valores neste ADR.
