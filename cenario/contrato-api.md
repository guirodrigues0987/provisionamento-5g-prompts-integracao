# Cenário: Provisionamento de dispositivos IoT em rede 5G privada

## Visão geral

Um operador usa o **Portal de Provisionamento** (front-end web) para cadastrar dispositivos IoT (sensores, câmeras, robôs) na rede 5G privada de um laboratório. O cadastro envolve três serviços:

```
[Portal Web] --> [API de Provisionamento] --> [Inventário de SIM]
                          |
                          +--------------> [Core 5G - cadastro de assinantes]
```

1. O operador preenche nome, ICCID do SIM, slice e perfil de QoS no portal.
2. O portal chama `POST /v1/dispositivos` na API de Provisionamento.
3. A API de Provisionamento reserva o SIM no Inventário e recebe o IMSI correspondente.
4. Com o IMSI, cadastra o assinante no Core 5G com o slice e o perfil de QoS.
5. Se tudo der certo, o dispositivo fica `ATIVO`. Se o Core falhar, o SIM é liberado (compensação).

## API de Provisionamento (serviço sob teste)

Autenticação: `Authorization: Bearer <JWT>`. Apenas usuários com papel `operador` ou `admin` podem criar e remover dispositivos. Qualquer usuário autenticado pode consultar.

### POST /v1/dispositivos

Headers obrigatórios: `Authorization`, `Idempotency-Key` (UUID).

Request:
```json
{
  "nome": "sensor-temp-bancada-02",
  "iccid": "8955010012345678901",
  "slice": "miot",
  "perfil_qos": "padrao"
}
```

Regras:
- `nome`: 3 a 40 caracteres, letras minúsculas, dígitos e hífen; único entre dispositivos ativos.
- `iccid`: 19 ou 20 dígitos, começando com `89`.
- `slice`: `embb`, `urllc` ou `miot`.
- `perfil_qos`: `padrao`, `baixa-latencia` ou `video`.
- `perfil_qos = baixa-latencia` só é aceito com `slice = urllc`.
- O slice `urllc` aceita no máximo 20 dispositivos ativos.
- Mesma `Idempotency-Key` com o mesmo corpo em até 24 h retorna a resposta original sem reprocessar. Mesma chave com corpo diferente retorna 409.

Respostas:

| Status | Código | Quando |
|---|---|---|
| 201 | — | Dispositivo criado e ativo |
| 202 | `PENDENTE_VERIFICACAO` | Core 5G não respondeu dentro de 3 s; resultado incerto |
| 400 | `VALIDACAO` | Campo ausente ou fora das regras (corpo traz `campos` com a lista dos inválidos) |
| 401 | `NAO_AUTENTICADO` | Token ausente, inválido ou expirado |
| 403 | `SEM_PERMISSAO` | Papel sem permissão |
| 404 | `SIM_NAO_ENCONTRADO` | ICCID não existe no inventário |
| 409 | `SIM_INDISPONIVEL` | SIM já reservado ou em uso |
| 409 | `NOME_DUPLICADO` | Já existe dispositivo ativo com esse nome |
| 409 | `LIMITE_SLICE` | Slice `urllc` já tem 20 dispositivos ativos |
| 409 | `IDEMPOTENCIA_CONFLITO` | Chave reutilizada com corpo diferente |
| 502 | `CORE_INDISPONIVEL` | Core 5G retornou erro 5xx (SIM é liberado) |
| 503 | `INVENTARIO_INDISPONIVEL` | Inventário fora do ar ou timeout (nada é criado) |

Resposta 201:
```json
{
  "id": "d3f1c2a0-7b1e-4c55-9a51-0f2a6c1b9e10",
  "nome": "sensor-temp-bancada-02",
  "iccid": "8955010012345678901",
  "imsi": "724990000012345",
  "slice": "miot",
  "perfil_qos": "padrao",
  "status": "ATIVO",
  "criado_em": "2026-10-22T14:03:11Z"
}
```

Corpo de erro (todos os 4xx/5xx):
```json
{ "codigo": "SIM_INDISPONIVEL", "mensagem": "O SIM informado já está em uso.", "campos": [] }
```

### GET /v1/dispositivos/{id}

Retorna o dispositivo (mesmo formato do 201). Status possíveis: `ATIVO`, `PENDENTE_VERIFICACAO`, `FALHA`, `REMOVIDO`. Retorna 404 `DISPOSITIVO_NAO_ENCONTRADO` se o id não existir.

Um job de reconciliação roda a cada 60 s: para dispositivos em `PENDENTE_VERIFICACAO`, consulta o Core. Se o assinante existir, muda para `ATIVO`; se não existir, muda para `FALHA` e libera o SIM.

### DELETE /v1/dispositivos/{id}

Remove o assinante do Core e libera o SIM no Inventário. Retorna 204. Se o Core falhar, retorna 502 e o dispositivo continua `ATIVO`.

## Inventário de SIM (dependência)

- `POST /v1/sims/{iccid}/reservar` → 200 `{ "iccid": "...", "imsi": "..." }` | 404 | 409
- `POST /v1/sims/{iccid}/liberar` → 204 | 404
- Timeout configurado na API de Provisionamento: 2 s.

## Core 5G — cadastro de assinantes (dependência)

- `POST /v1/assinantes` com `{ "imsi", "sst", "perfil_qos" }` → 201 | 409 (IMSI já cadastrado) | 5xx
- `GET /v1/assinantes/{imsi}` → 200 | 404
- `DELETE /v1/assinantes/{imsi}` → 204 | 404 | 5xx
- Mapeamento de slice para SST: `embb` → 1, `urllc` → 2, `miot` → 3.
- Timeout configurado na API de Provisionamento: 3 s, sem retry.

## Portal Web (front-end)

- Formulário com os quatro campos e botão "Provisionar".
- O botão fica desabilitado enquanto a requisição está em andamento.
- Cada clique gera uma nova `Idempotency-Key`; um reenvio automático (ex.: queda de conexão) reaproveita a mesma chave.
- Exibe mensagens a partir do `codigo` retornado pela API e destaca os campos listados em `campos`.
- A tela de detalhes do dispositivo mostra o status e, para `PENDENTE_VERIFICACAO`, atualiza sozinha a cada 15 s.
