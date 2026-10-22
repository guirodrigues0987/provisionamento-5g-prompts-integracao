# Saída — Prompt 1, versão simples

> Saída da LLM para `prompts/01-cenarios-simples.md`, sem edição. Observações de revisão no fim do arquivo.

## Casos de Teste de Integração — API de Provisionamento IoT 5G

| ID | Caso de teste | Passos | Resultado esperado |
|---|---|---|---|
| TC01 | Cadastrar dispositivo válido | Enviar POST /devices com IMEI, nome e tipo | 201 Created com ID do dispositivo |
| TC02 | Cadastrar dispositivo com IMEI inválido | POST /devices com IMEI de 10 dígitos | 400 Bad Request |
| TC03 | Cadastrar dispositivo duplicado | POST /devices com IMEI já cadastrado | 409 Conflict |
| TC04 | Consultar dispositivo existente | GET /devices/{id} | 200 com dados do dispositivo |
| TC05 | Consultar dispositivo inexistente | GET /devices/{id} com ID aleatório | 404 Not Found |
| TC06 | Listar dispositivos | GET /devices | 200 com lista paginada |
| TC07 | Ativar dispositivo | POST /devices/{id}/activate | 200, status "active" |
| TC08 | Atualizar perfil de QoS | PATCH /devices/{id} com novo QoS | 200 com QoS atualizado |
| TC09 | Remover dispositivo | DELETE /devices/{id} | 204 No Content |
| TC10 | Requisição sem token | POST /devices sem Authorization | 401 Unauthorized |
| TC11 | Integração com HSS/UDM | Cadastrar dispositivo e verificar no UDM | Assinante criado no UDM |
| TC12 | Atribuição de IP | Ativar dispositivo e verificar sessão PDU | Dispositivo recebe IP do SMF |

**Recomendações:** automatizar com Postman/Newman ou RestAssured, usar ambiente de homologação com core 5G simulado (ex.: Open5GS) e validar logs do AMF e SMF.

---

## Revisão (feita por mim)

- **12 casos**, mas só **2** (TC11 e TC12) envolvem mais de um componente. Os outros 10 testam um endpoint sozinho.
- **4 casos dependem de coisas que não existem no sistema:** listagem paginada (TC06), rota `/activate` (TC07), PATCH de QoS (TC08) e sessão PDU/IP (TC12). O modelo preencheu o vazio com o que é comum em APIs 5G genéricas.
- Usa IMEI como identificador, quando o sistema trabalha com ICCID/IMSI.
- Nenhum caso cobre idempotência, compensação, limite de slice ou o que o Portal mostra.
- Os resultados esperados param no status HTTP; não dizem nada sobre o estado do Inventário ou do Core.
