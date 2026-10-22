# Saída — Prompt 1, versão refinada

> Saída da LLM para `prompts/01-cenarios-refinado.md`. Observações de revisão no fim do arquivo.

## Resumo

| ID | Título | Componentes | Tipo |
|---|---|---|---|
| CT-INT-001 | Provisionamento completo com slice miot | API, Inventário, Core | Fluxo |
| CT-INT-002 | Mapeamento slice → SST para os três slices | API, Core | Mapeamento |
| CT-INT-003 | IMSI do Inventário é o IMSI cadastrado no Core | API, Inventário, Core | Mapeamento |
| CT-INT-004 | Ciclo criar → consultar → remover | API, Inventário, Core | Fluxo |
| CT-INT-005 | SIM liberado na remoção pode ser reutilizado | API, Inventário, Core | Estado |
| CT-INT-006 | Nome liberado após remoção pode ser reutilizado | API, Core | Estado |
| CT-INT-007 | 20º dispositivo urllc aceito, 21º recusado | API, Core | Estado |
| CT-INT-008 | Vaga urllc liberada após remoção | API, Core | Estado |
| CT-INT-009 | Reenvio com mesma Idempotency-Key não duplica | Portal, API, Inventário, Core | Idempotência |
| CT-INT-010 | Pendente vira ATIVO na reconciliação | API, Core | Reconciliação |
| CT-INT-011 | Pendente vira FALHA e libera SIM | API, Inventário, Core | Reconciliação |
| CT-INT-012 | Portal exibe cada resultado do fluxo | Portal, API | Tela |
| CT-INV-101 | Dispositivo FALHA conta para nome único? | API | INVESTIGAR |
| CT-INV-102 | DELETE em dispositivo PENDENTE_VERIFICACAO | API, Core | INVESTIGAR |
| CT-INV-103 | Idempotency-Key após 24 h | API | INVESTIGAR |

---

### CT-INT-001 — Provisionamento completo com slice miot

**Componentes:** API de Provisionamento, Inventário de SIM, Core 5G
**Pré-condições:** SIM `8955010012345678901` livre no Inventário, com IMSI `724990000012345`; nenhum dispositivo ativo chamado `sensor-temp-bancada-02`; token com papel `operador`.

**Passos**
1. `POST /v1/dispositivos` com `Idempotency-Key: 6f1d...` e corpo:
   ```json
   { "nome": "sensor-temp-bancada-02", "iccid": "8955010012345678901", "slice": "miot", "perfil_qos": "padrao" }
   ```
2. `GET /v1/dispositivos/{id}` com o id retornado.
3. Consultar `GET /v1/assinantes/724990000012345` no Core.
4. Tentar `POST /v1/sims/8955010012345678901/reservar` direto no Inventário.

**Resultado esperado**
- API: 201, `status = ATIVO`, `imsi = 724990000012345`. O GET devolve o mesmo corpo.
- Core: 200, `sst = 3`, `perfil_qos = padrao`.
- Inventário: 409 na nova reserva (o SIM está em uso).

### CT-INT-002 — Mapeamento slice → SST para os três slices

**Componentes:** API, Core
**Pré-condições:** três SIMs livres; menos de 20 dispositivos urllc ativos.

**Passos**
1. Provisionar `cam-portaria-01` com `slice = embb`, `perfil_qos = video`.
2. Provisionar `cobot-celula-01` com `slice = urllc`, `perfil_qos = baixa-latencia`.
3. Provisionar `sensor-umid-01` com `slice = miot`, `perfil_qos = padrao`.
4. Consultar cada IMSI no Core.

**Resultado esperado**
- Core: `sst` 1, 2 e 3, respectivamente, e o `perfil_qos` idêntico ao enviado.

### CT-INT-003 — IMSI do Inventário é o IMSI cadastrado no Core

**Componentes:** API, Inventário, Core
**Pré-condições:** SIM `8955010099990000011` livre, associado no Inventário ao IMSI `724990000099911` (valor sem relação aparente com o ICCID).

**Passos**
1. Provisionar dispositivo com esse ICCID.
2. Consultar `GET /v1/assinantes/724990000099911` no Core.

**Resultado esperado**
- API: campo `imsi = 724990000099911`.
- Core: assinante existe com esse IMSI. Garante que a API usa o valor devolvido pelo Inventário e não deriva o IMSI do ICCID.

### CT-INT-004 — Ciclo criar → consultar → remover

**Componentes:** API, Inventário, Core
**Pré-condições:** as mesmas do CT-INT-001.

**Passos**
1. Provisionar o dispositivo (201).
2. `DELETE /v1/dispositivos/{id}`.
3. `GET /v1/dispositivos/{id}`.
4. `GET /v1/assinantes/{imsi}` no Core.
5. `POST /v1/sims/{iccid}/reservar` no Inventário e, em seguida, `POST /v1/sims/{iccid}/liberar` para restaurar o ambiente.

**Resultado esperado**
- DELETE: 204. GET: `status = REMOVIDO`.
- Core: 404 para o IMSI.
- Inventário: reserva aceita (200), o SIM voltou a ficar livre.

### CT-INT-005 — SIM liberado na remoção pode ser reutilizado

**Componentes:** API, Inventário, Core
**Pré-condições:** dispositivo `sensor-a` ativo com o SIM `8955010012345678901`.

**Passos**
1. `DELETE` do `sensor-a`.
2. Provisionar `sensor-b` com o mesmo ICCID.

**Resultado esperado**
- API: 201 para `sensor-b`, com o mesmo IMSI.
- Core: assinante existe uma única vez para esse IMSI.

### CT-INT-006 — Nome liberado após remoção pode ser reutilizado

**Componentes:** API, Core
**Pré-condições:** dispositivo `robo-agv-01` ativo; outro SIM livre.

**Passos**
1. `DELETE` do `robo-agv-01`.
2. Provisionar novo `robo-agv-01` com o outro SIM.

**Resultado esperado**
- 201. A regra de nome único vale só entre dispositivos ativos.

### CT-INT-007 — 20º dispositivo urllc aceito, 21º recusado

**Componentes:** API, Core
**Pré-condições:** exatamente 19 dispositivos urllc ativos; dois SIMs livres.

**Passos**
1. Provisionar `cobot-20` com `slice = urllc`.
2. Provisionar `cobot-21` com `slice = urllc`.
3. Consultar o IMSI do SIM usado no passo 2 no Core.

**Resultado esperado**
- Passo 1: 201.
- Passo 2: 409 `LIMITE_SLICE`.
- Core: 404 para o IMSI do passo 2. Inventário: SIM do passo 2 continua livre.

### CT-INT-008 — Vaga urllc liberada após remoção

**Componentes:** API, Core
**Pré-condições:** 20 dispositivos urllc ativos.

**Passos**
1. `DELETE` de um dispositivo urllc.
2. Provisionar novo dispositivo urllc.

**Resultado esperado**
- 201. O contador de vagas considera só dispositivos ativos.

### CT-INT-009 — Reenvio com mesma Idempotency-Key não duplica

**Componentes:** Portal, API, Inventário, Core
**Pré-condições:** SIM livre; proxy entre Portal e API configurado para descartar a primeira resposta (simula queda de conexão depois que a API processou).

**Passos**
1. No Portal, preencher o formulário e clicar em "Provisionar".
2. O Portal reenvia automaticamente com a mesma `Idempotency-Key`.
3. Consultar no Core quantas vezes o IMSI foi cadastrado (log de requisições do stub ou contagem).

**Resultado esperado**
- API: a segunda chamada retorna 201 com o mesmo `id` da primeira.
- Core: uma única chamada `POST /v1/assinantes`.
- Portal: mostra a tela de sucesso uma vez, sem mensagem de erro.

### CT-INT-010 — Pendente vira ATIVO na reconciliação

**Componentes:** API, Core
**Pré-condições:** stub do Core configurado para criar o assinante mas responder depois de 4 s.

**Passos**
1. Provisionar dispositivo.
2. Aguardar até 60 s (um ciclo do job) e consultar `GET /v1/dispositivos/{id}`.

**Resultado esperado**
- Passo 1: 202 `PENDENTE_VERIFICACAO`.
- Passo 2: `status = ATIVO`. Inventário: SIM continua reservado.

### CT-INT-011 — Pendente vira FALHA e libera SIM

**Componentes:** API, Inventário, Core
**Pré-condições:** stub do Core configurado para não criar o assinante e não responder em 3 s.

**Passos**
1. Provisionar dispositivo.
2. Aguardar até 60 s e consultar o dispositivo.
3. Tentar reservar o SIM direto no Inventário.

**Resultado esperado**
- Passo 1: 202. Passo 2: `status = FALHA`.
- Inventário: reserva aceita (SIM foi liberado pelo job).

### CT-INT-012 — Portal exibe cada resultado do fluxo

**Componentes:** Portal, API
**Pré-condições:** API apontando para stubs que permitem forçar cada resposta.

**Passos e resultado esperado**

| Resposta forçada | O Portal deve |
|---|---|
| 201 | Ir para a tela de detalhes com status ATIVO |
| 202 `PENDENTE_VERIFICACAO` | Ir para a tela de detalhes e atualizar sozinha a cada 15 s até mudar o status |
| 400 com `campos: ["iccid"]` | Manter o formulário preenchido e destacar só o campo ICCID |
| 409 `LIMITE_SLICE` | Mostrar a mensagem do código, sem destacar campos |
| 503 `INVENTARIO_INDISPONIVEL` | Mostrar erro e reabilitar o botão "Provisionar" |

---

### CT-INV-101 — Dispositivo em FALHA conta para nome único?

**Pergunta para o time:** o contrato diz "único entre dispositivos ativos". Um dispositivo em `FALHA` ou `PENDENTE_VERIFICACAO` bloqueia o nome? Se `PENDENTE` não bloquear, dois dispositivos podem acabar ativos com o mesmo nome depois da reconciliação.

### CT-INV-102 — DELETE em dispositivo PENDENTE_VERIFICACAO

**Pergunta para o time:** o contrato só descreve o DELETE de dispositivo `ATIVO`. O que acontece se o operador remover enquanto o job ainda não reconciliou? O job pode reativar algo que o operador acabou de remover.

### CT-INV-103 — Idempotency-Key após 24 h

**Pergunta para o time:** depois de 24 h, a mesma chave com o mesmo corpo é tratada como requisição nova? Se sim, o retorno esperado é 409 `SIM_INDISPONIVEL` (SIM já em uso pelo próprio dispositivo)?

---

## Revisão (feita por mim)

- **15 casos**, todos com pelo menos dois componentes. 12 executáveis e 3 viraram perguntas para o time.
- Nenhum endpoint ou código inventado. Os valores de teste (ICCID, IMSI, nomes) seguem as regras do contrato.
- Ponto de atenção na revisão: o CT-INT-009 só faz sentido porque o contrato descreve o reenvio automático do Portal. Um teste de "duplo clique" não pegaria nada, já que o botão fica desabilitado e cada clique gera chave nova.
- Ainda não cobre falhas (vem do prompt 2) nem os papéis de usuário além de `operador`.
