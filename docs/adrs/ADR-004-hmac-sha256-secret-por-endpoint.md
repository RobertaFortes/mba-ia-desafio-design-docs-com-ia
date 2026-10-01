# ADR-004: Autenticação por HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:22] Sofia)
- **Decisores:** Sofia, Larissa, Diego
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

Os eventos carregam dados de pedidos e saem para endpoints fora da nossa infraestrutura. O cliente precisa validar que a requisição veio de nós e que o payload não foi adulterado ([09:19] Sofia). Já houve cliente que vazou secret em log da própria aplicação ([09:22] Diego).

## Decisão

- Assinar o **corpo do request** com **HMAC-SHA256**, enviado no header `X-Signature` ([09:20] Sofia).
- **Uma secret única por endpoint de webhook** (não global). A secret é gerada por nós e devolvida na criação ([09:21] Sofia, [09:31] Marcos).
- A secret é **rotacionável pela API**; na rotação, a **antiga permanece válida por 24 horas** em paralelo e depois deixa de valer ([09:21] Sofia).
- A tabela de configuração guarda url, secret, customer_id e estado ativo ([09:21] Bruno/Sofia).
- A URL deve ser `https`; `http` é recusado por validação no schema Zod ([09:23] Sofia). Isso é validação, não decisão arquitetural; fica aqui apenas como contexto.
- A revisão de segurança (HMAC e geração de secret) é pré-requisito do deploy, com 2 dias úteis reservados ([09:46] Sofia).

## Alternativas Consideradas

1. **Secret global da plataforma.** Descartada: o vazamento de uma secret comprometeria todos os clientes ([09:21] Sofia).
2. **Rotação com corte imediato da secret antiga.** Descartada: o cliente precisa de tempo para migrar seus sistemas ([09:21] Sofia).
3. **Sem assinatura, apenas HTTPS.** Não discutida como opção formal; HTTPS não prova origem nem integridade fim a fim para o consumidor, e a reunião tratou HMAC como padrão de mercado ([09:20] Sofia). Registrada como alternativa plausível.

## Consequências

**Positivas**
- Vazamento é contido a um endpoint; rotação sem downtime para o cliente.
- Padrão de mercado, com bibliotecas disponíveis do lado do cliente.

**Negativas / trade-offs**
- Como o HMAC exige a secret em claro no momento de assinar, ela precisa ser armazenada de forma recuperável (não dá para guardar só um hash). A forma de proteção em repouso **não foi discutida** e vai para a revisão da Sofia (questão em aberto no RFC).
- Durante o grace period existem duas secrets válidas por endpoint, o que aumenta a complexidade do modelo e da assinatura (como o `X-Signature` carrega ambas é detalhe aberto, ver RFC/FDD).
- `X-Signature` cobre o corpo; `X-Timestamp` é enviado separadamente para detecção de replay pelo cliente ([09:44] Diego), mas não faz parte do conteúdo assinado segundo o que foi dito na reunião.
