# Prompt 2 — Casos de erro e falha (versão refinada)

Junto com este prompt foi colado o conteúdo inteiro de `cenario/contrato-api.md`.

```
Você é um analista de QA responsável pelos testes de resiliência de uma integração entre
microsserviços.

CONTEXTO
<contrato>
[conteúdo de cenario/contrato-api.md]
</contrato>

OBJETIVO
Gerar casos de teste de ERRO e FALHA na integração do fluxo POST /v1/dispositivos e
DELETE /v1/dispositivos/{id}. O foco é o que acontece com o estado dos três serviços
quando algo dá errado no meio do caminho, não apenas o status HTTP devolvido.

COBRIR OBRIGATORIAMENTE
1. Falhas do Inventário de SIM: 404, 409, timeout acima de 2 s, serviço fora do ar.
2. Falhas do Core 5G: 5xx, 409 (IMSI já cadastrado), timeout acima de 3 s.
3. Compensação: o SIM é liberado quando deve ser e mantido quando deve ser.
4. Falhas na remoção (DELETE) e o estado que fica em cada serviço.
5. Concorrência: duas requisições disputando o mesmo SIM, o mesmo nome ou a última vaga
   do slice urllc.
6. Idempotência em situação de erro (reenvio após timeout, chave reaproveitada com corpo
   diferente).
7. Autenticação e autorização no fluxo completo (token expirado, papel sem permissão).
8. O que o Portal Web exibe para cada código de erro e quais campos ele destaca.

REGRAS
- Use somente códigos e campos do contrato. O que o contrato não definir vira caso
  "INVESTIGAR" com a pergunta para o time.
- Para simular falhas das dependências, indique o mecanismo: stub/mock configurado para
  devolver o status, ou atraso injetado (ex.: WireMock com fixedDelayMilliseconds). Nada de
  "requisições chegam no mesmo milissegundo"; para concorrência, use disparo paralelo com
  barreira (ex.: k6 ou script com threads) e descreva o que deve ser verdade no final,
  independente da ordem.
- Inclua o payload exato de cada requisição.

FORMATO DE SAÍDA
Markdown. Tabela-resumo no início (ID, título, dependência que falha, código esperado) e
depois cada caso com:
- ID no padrão CT-ERR-NNN (ou CT-INV-NNN para casos INVESTIGAR, continuando a numeração 1xx)
- Título
- Mecanismo de simulação
- Pré-condições
- Requisição (headers relevantes + corpo)
- Resultado esperado: resposta da API, estado no Inventário, estado no Core, tela do Portal
```
