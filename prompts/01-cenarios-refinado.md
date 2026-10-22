# Prompt 1 — Cenários de integração (versão refinada)

Junto com este prompt foi colado o conteúdo inteiro de `cenario/contrato-api.md`.

```
Você é um analista de QA especializado em testes de integração de sistemas distribuídos.

CONTEXTO
O sistema abaixo provisiona dispositivos IoT numa rede 5G privada. Envolve quatro componentes:
Portal Web, API de Provisionamento, Inventário de SIM e Core 5G. O contrato completo está
entre as tags <contrato>.

<contrato>
[conteúdo de cenario/contrato-api.md]
</contrato>

OBJETIVO
Gerar casos de teste de INTEGRAÇÃO para o fluxo de provisionamento, ou seja, casos em que
pelo menos dois componentes interagem. Não gere testes de um endpoint isolado (validação de
campo, status de resposta sozinho) — esses ficam para outro prompt.

COBRIR OBRIGATORIAMENTE
1. Fluxos encadeados ponta a ponta (criar → consultar → remover) e o estado resultante em
   cada um dos três serviços.
2. Mapeamento de dados entre serviços: ICCID → IMSI vindo do Inventário, slice → SST enviado
   ao Core, perfil de QoS repassado sem alteração.
3. Regras que dependem de estado compartilhado: limite de 20 dispositivos no slice urllc,
   nome único entre ativos, reaproveitamento de SIM após remoção.
4. Idempotência fim a fim, incluindo o reenvio automático feito pelo Portal.
5. O job de reconciliação e a transição de PENDENTE_VERIFICACAO para ATIVO ou FALHA.
6. O que o Portal Web deve exibir em cada resultado do fluxo.

REGRAS
- Use somente endpoints, campos, códigos e valores que existem no contrato.
- Quando o contrato não definir um comportamento necessário para o teste, NÃO invente:
  crie o caso com tipo "INVESTIGAR" e escreva a pergunta que o time precisa responder.
- Cada verificação de resultado deve ser observável: status HTTP, campo de resposta,
  consulta ao Inventário ou ao Core, ou elemento de tela.

FORMATO DE SAÍDA
Markdown. Uma tabela-resumo no início (ID, título, componentes envolvidos, tipo) e depois
cada caso com:
- ID no padrão CT-INT-NNN (ou CT-INV-NNN para casos INVESTIGAR)
- Título
- Componentes envolvidos
- Pré-condições
- Passos (numerados, com requisição quando houver)
- Resultado esperado em cada serviço afetado
```
