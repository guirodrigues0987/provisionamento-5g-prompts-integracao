# Saída — Prompt 2, versão simples

> Saída da LLM para `prompts/02-erros-simples.md`, sem edição. Observações de revisão no fim do arquivo.

## Casos de teste de erro — Integração da API de Provisionamento 5G

1. **Serviço dependente indisponível** — derrubar o serviço dependente e enviar requisição. Esperado: 503 Service Unavailable com mensagem amigável.
2. **Timeout na dependência** — simular lentidão na resposta. Esperado: 504 Gateway Timeout.
3. **Resposta malformada da dependência** — dependência retorna JSON inválido. Esperado: 502 Bad Gateway e log de erro.
4. **Token expirado** — enviar requisição com JWT expirado. Esperado: 401.
5. **Usuário sem permissão** — usuário comum tenta provisionar. Esperado: 403.
6. **Payload vazio** — POST sem corpo. Esperado: 400.
7. **Campos obrigatórios ausentes** — omitir IMSI. Esperado: 400 com lista de campos.
8. **IMSI duplicado** — provisionar IMSI já existente. Esperado: 409.
9. **Rate limiting** — enviar 1000 requisições por minuto. Esperado: 429 Too Many Requests.
10. **Falha no banco de dados** — derrubar o banco. Esperado: 500 e rollback da transação.
11. **Certificado TLS inválido na dependência** — Esperado: falha de conexão tratada e logada.
12. **Falha parcial no provisionamento** — dependência A responde e B falha. Esperado: rollback das operações já realizadas.
13. **Atualização concorrente** — dois PATCH no mesmo dispositivo. Esperado: um deles recebe 409.

**Boas práticas:** usar WireMock ou Mountebank para simular dependências, aplicar circuit breaker e garantir logs com correlation ID.

---

## Revisão (feita por mim)

- **13 itens**, mas é um checklist que serviria para quase qualquer API. Só o item 12 trata de estado entre serviços, e de forma genérica ("dependência A e B").
- **5 itens esperam comportamentos que o contrato não tem:** 504 (o contrato devolve 202 no timeout do Core e 503 no do Inventário), 429/rate limiting, 500 de banco, PATCH concorrente e IMSI enviado pelo cliente (o cliente envia ICCID).
- Nenhum caso traz payload pronto para executar.
- Não diz qual dependência falha em cada caso, então não dá para saber se o SIM deveria ser liberado ou não.
- Nada sobre idempotência, que é justamente o risco maior quando há timeout.
