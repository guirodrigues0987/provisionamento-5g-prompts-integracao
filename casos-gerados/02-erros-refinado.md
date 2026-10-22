# Saída — Prompt 2, versão refinada

> Saída da LLM para `prompts/02-erros-refinado.md`. Observações de revisão no fim do arquivo.

**Payload base** (usado quando o caso não indicar outro):
```http
POST /v1/dispositivos
Authorization: Bearer <token-operador>
Idempotency-Key: <uuid novo>
Content-Type: application/json

{ "nome": "sensor-temp-bancada-02", "iccid": "8955010012345678901", "slice": "miot", "perfil_qos": "padrao" }
```

## Resumo

| ID | Título | Dependência que falha | Código esperado |
|---|---|---|---|
| CT-ERR-001 | ICCID inexistente no Inventário | Inventário (404) | 404 SIM_NAO_ENCONTRADO |
| CT-ERR-002 | SIM já reservado | Inventário (409) | 409 SIM_INDISPONIVEL |
| CT-ERR-003 | Inventário com timeout | Inventário (> 2 s) | 503 INVENTARIO_INDISPONIVEL |
| CT-ERR-004 | Inventário fora do ar | Inventário (conexão recusada) | 503 INVENTARIO_INDISPONIVEL |
| CT-ERR-005 | Core retorna 500 | Core (5xx) | 502 CORE_INDISPONIVEL |
| CT-ERR-006 | Core com timeout | Core (> 3 s) | 202 PENDENTE_VERIFICACAO |
| CT-ERR-007 | Falha ao liberar SIM na compensação | Core (5xx) + Inventário (liberar falha) | 502 CORE_INDISPONIVEL |
| CT-ERR-008 | DELETE com Core em 5xx | Core (5xx) | 502 CORE_INDISPONIVEL |
| CT-ERR-009 | DELETE com assinante já ausente no Core | Core (404) | 204 |
| CT-ERR-010 | Dois provisionamentos com o mesmo SIM | Concorrência | 201 + 409 SIM_INDISPONIVEL |
| CT-ERR-011 | Dois provisionamentos com o mesmo nome | Concorrência | 201 + 409 NOME_DUPLICADO |
| CT-ERR-012 | Disputa pela última vaga urllc | Concorrência | 201 + 409 LIMITE_SLICE |
| CT-ERR-013 | Reenvio após 202 com a mesma chave | Core (> 3 s) | 202 repetido |
| CT-ERR-014 | Mesma chave com corpo diferente | — | 409 IDEMPOTENCIA_CONFLITO |
| CT-ERR-015 | Token expirado | — | 401 NAO_AUTENTICADO |
| CT-ERR-016 | Papel sem permissão | — | 403 SEM_PERMISSAO |
| CT-ERR-017 | baixa-latencia fora do urllc | — | 400 VALIDACAO |
| CT-INV-104 | Core retorna 409 (IMSI já cadastrado) | Core (409) | não definido |
| CT-INV-105 | 401/403 no meio do formulário | — | não definido |
| CT-INV-106 | Compensação que falha | Inventário (liberar) | não definido |

---

### CT-ERR-001 — ICCID inexistente no Inventário

**Mecanismo:** dado real; ICCID `8955010000000000000` não cadastrado no Inventário.
**Requisição:** payload base com esse ICCID.
**Resultado esperado**
- API: 404 `SIM_NAO_ENCONTRADO`.
- Core: nenhuma chamada recebida.
- Portal: mensagem do código e campo ICCID destacado.

### CT-ERR-002 — SIM já reservado

**Mecanismo:** dado real; SIM reservado previamente via `POST /v1/sims/{iccid}/reservar`.
**Requisição:** payload base.
**Resultado esperado**
- API: 409 `SIM_INDISPONIVEL`.
- Inventário: reserva anterior intacta (não é liberada pela tentativa).
- Core: nenhuma chamada.

### CT-ERR-003 — Inventário com timeout

**Mecanismo:** WireMock na frente do Inventário com `fixedDelayMilliseconds: 2500` em `/reservar`.
**Requisição:** payload base.
**Resultado esperado**
- API: 503 `INVENTARIO_INDISPONIVEL` em até ~2 s.
- Core: nenhuma chamada.
- Inventário: verificar depois de 3 s se o SIM ficou reservado (a reserva pode ter sido concluída no Inventário mesmo com a API tendo desistido). Ver CT-INV-106.
- Portal: mensagem de erro e botão reabilitado.

### CT-ERR-004 — Inventário fora do ar

**Mecanismo:** container do Inventário parado (conexão recusada).
**Requisição:** payload base.
**Resultado esperado**
- API: 503 `INVENTARIO_INDISPONIVEL`. Nenhum dispositivo criado (`GET` pelo nome não retorna nada ativo).
- Core: nenhuma chamada.

### CT-ERR-005 — Core retorna 500

**Mecanismo:** stub do Core devolvendo 500 em `POST /v1/assinantes`.
**Requisição:** payload base.
**Resultado esperado**
- API: 502 `CORE_INDISPONIVEL`.
- Inventário: o SIM foi liberado (`/reservar` direto aceita).
- Core: nenhum assinante para o IMSI.
- Portal: mensagem do código; formulário mantém os dados para nova tentativa.

### CT-ERR-006 — Core com timeout

**Mecanismo:** WireMock no Core com `fixedDelayMilliseconds: 4000`, criando o assinante normalmente.
**Requisição:** payload base.
**Resultado esperado**
- API: 202 `PENDENTE_VERIFICACAO` em ~3 s.
- Inventário: SIM continua reservado (não pode liberar, o resultado é incerto).
- Portal: tela de detalhes com status pendente e atualização a cada 15 s.

### CT-ERR-007 — Falha ao liberar SIM na compensação

**Mecanismo:** stub do Core com 500 em `POST /v1/assinantes` e stub do Inventário com 500 em `/liberar`.
**Requisição:** payload base.
**Resultado esperado**
- API: 502 `CORE_INDISPONIVEL`.
- Inventário: SIM fica preso como reservado sem dispositivo associado. O teste registra o estado e alimenta o CT-INV-106.

### CT-ERR-008 — DELETE com Core em 5xx

**Mecanismo:** dispositivo ativo; stub do Core devolvendo 503 em `DELETE /v1/assinantes/{imsi}`.
**Requisição:** `DELETE /v1/dispositivos/{id}` com token `operador`.
**Resultado esperado**
- API: 502. `GET` do dispositivo: `status = ATIVO`.
- Inventário: SIM continua em uso (não foi liberado antes da remoção no Core).
- Core: assinante continua existindo.

### CT-ERR-009 — DELETE com assinante já ausente no Core

**Mecanismo:** dispositivo ativo; assinante removido manualmente do Core antes do teste.
**Requisição:** `DELETE /v1/dispositivos/{id}`.
**Resultado esperado**
- API: 204 (o 404 do Core é tratado como "já removido"). Status `REMOVIDO`.
- Inventário: SIM liberado.

### CT-ERR-010 — Dois provisionamentos com o mesmo SIM

**Mecanismo:** script k6 com dois VUs e barreira (`exec.scenario.iterationInTest` sincronizado), cada um com nome diferente e o mesmo ICCID, chaves de idempotência diferentes.
**Resultado esperado** (independente da ordem)
- Exatamente uma resposta 201 e uma 409 `SIM_INDISPONIVEL`.
- Core: exatamente um assinante para o IMSI.
- API: exatamente um dispositivo ativo com esse ICCID.

### CT-ERR-011 — Dois provisionamentos com o mesmo nome

**Mecanismo:** mesmo script, mesmo `nome`, ICCIDs diferentes.
**Resultado esperado**
- Uma 201 e uma 409 `NOME_DUPLICADO`.
- Inventário: o SIM da requisição recusada está livre.
- Core: nenhum assinante para o IMSI da requisição recusada.

### CT-ERR-012 — Disputa pela última vaga urllc

**Mecanismo:** 19 dispositivos urllc ativos; script dispara 3 requisições urllc em paralelo com SIMs e nomes diferentes.
**Resultado esperado**
- Exatamente uma 201 e duas 409 `LIMITE_SLICE`.
- Contagem final de dispositivos urllc ativos: 20.
- Inventário: os dois SIMs recusados estão livres.

### CT-ERR-013 — Reenvio após 202 com a mesma chave

**Mecanismo:** como no CT-ERR-006; reenviar a mesma requisição, com a mesma `Idempotency-Key`, logo após o 202.
**Resultado esperado**
- Segunda resposta: 202 com o mesmo `id`.
- Core: uma única chamada `POST /v1/assinantes` no log do stub.

### CT-ERR-014 — Mesma chave com corpo diferente

**Mecanismo:** dado real.
**Requisição:** payload base com a chave `K1` (201); depois, mesma chave `K1` com `nome = sensor-temp-bancada-03`.
**Resultado esperado**
- Segunda resposta: 409 `IDEMPOTENCIA_CONFLITO`.
- Inventário e Core: nenhuma chamada nova.

### CT-ERR-015 — Token expirado

**Mecanismo:** JWT com `exp` no passado.
**Resultado esperado**
- API: 401 `NAO_AUTENTICADO`.
- Inventário e Core: nenhuma chamada (a autenticação acontece antes de qualquer reserva).

### CT-ERR-016 — Papel sem permissão

**Mecanismo:** JWT válido com papel `leitura`.
**Requisição:** payload base; depois `DELETE` de um dispositivo existente.
**Resultado esperado**
- Ambas: 403 `SEM_PERMISSAO`.
- Inventário e Core: nenhuma chamada. Dispositivo do DELETE continua ativo.

### CT-ERR-017 — baixa-latencia fora do urllc

**Mecanismo:** dado real.
**Requisição:** payload base com `perfil_qos = baixa-latencia` e `slice = miot`.
**Resultado esperado**
- API: 400 `VALIDACAO` com `campos` contendo `perfil_qos` (e/ou `slice`).
- Inventário: SIM não reservado. Core: nenhuma chamada.
- Portal: campos listados destacados.

---

### CT-INV-104 — Core retorna 409 (IMSI já cadastrado)

**Pergunta para o time:** o contrato não diz como a API trata o 409 do Core. Isso acontece se um assinante ficou "órfão" (ex.: CT-ERR-007 ou timeout sem reconciliação). A API deve considerar sucesso, devolver erro ou liberar o SIM?

### CT-INV-105 — Token expira com o formulário aberto

**Pergunta para o time:** quando a API devolve 401 ou 403, o Portal redireciona para login, mostra mensagem ou perde os dados do formulário? O contrato descreve o comportamento do Portal só para os códigos de negócio.

### CT-INV-106 — Compensação que falha

**Pergunta para o time:** se a liberação do SIM falhar (CT-ERR-007) ou a reserva concluir depois do timeout (CT-ERR-003), quem limpa o SIM preso? O job de reconciliação só olha dispositivos `PENDENTE_VERIFICACAO`, então esses SIMs ficariam reservados para sempre.

---

## Revisão (feita por mim)

- **20 casos**: 17 executáveis, todos com payload e mecanismo de simulação, e 3 perguntas para o time.
- Os casos mais valiosos são os que verificam o estado das dependências depois do erro (CT-ERR-005, 007, 008, 010 a 012). Foi isso que o modelo deixou de fazer na versão simples.
- As regras "indicar o mecanismo de simulação" e "só códigos do contrato" entraram no prompt por causa da versão simples, que esperava 504 no timeout e falava em concorrência sem dizer como reproduzir. Na refinada, nenhum caso ficou sem mecanismo e o timeout do Core aparece corretamente como 202.
- O CT-INV-106 achou um buraco real no desenho: SIM preso sem dono quando a compensação falha.
