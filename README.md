# Prompts para testes de integração com GenAI — Provisionamento IoT em 5G privada

Atividade do fórum da Unidade III: usar uma LLM como apoio na criação de casos de teste de integração e avaliar como a forma do prompt muda o resultado.

## Cenário

Um portal web cadastra dispositivos IoT (sensores, câmeras, robôs) numa rede 5G privada. A API de Provisionamento reserva o SIM no Inventário, recebe o IMSI e cadastra o assinante no Core 5G com o slice e o perfil de QoS. Se o Core falha, o SIM é liberado. Se o Core demora, o dispositivo fica pendente até um job de reconciliação resolver.

É um bom cenário de integração porque o erro raramente está num serviço isolado: está no estado que sobra entre eles quando algo falha no meio do caminho.

Contrato completo: [`cenario/contrato-api.md`](cenario/contrato-api.md)

## Os dois tipos de prompt

| Tipo | Objetivo | Versão simples | Versão refinada |
|---|---|---|---|
| 1. Cenários de integração | Fluxos encadeados entre os serviços e o estado em cada um | [prompt](prompts/01-cenarios-simples.md) · [saída](casos-gerados/01-cenarios-simples.md) | [prompt](prompts/01-cenarios-refinado.md) · [saída](casos-gerados/01-cenarios-refinado.md) |
| 2. Casos de erro e falha | Falhas das dependências, compensação, concorrência e idempotência | [prompt](prompts/02-erros-simples.md) · [saída](casos-gerados/02-erros-simples.md) | [prompt](prompts/02-erros-refinado.md) · [saída](casos-gerados/02-erros-refinado.md) |

Cada tipo foi rodado em duas versões: uma de uma linha e uma refinada, com papel, contrato da API colado, lista do que cobrir, regras e formato de saída. As saídas foram geradas com o Claude e revisadas contra o contrato; a revisão está no fim de cada arquivo de saída.

## Resultado

| | Cenários simples | Cenários refinado | Erros simples | Erros refinado |
|---|---|---|---|---|
| Casos gerados | 12 | 15 | 13 | 20 |
| Envolvem 2+ componentes | 2 | 15 | 1 | 20 |
| Dependem de algo que não existe no sistema | 4 | 0 | 5 | 0 |
| Verificam o estado das dependências | 0 | 9 | 0 | 17 |
| Com payload pronto | 0 | parte | 0 | 17 |
| Perguntas para o time (INVESTIGAR) | 0 | 3 | 0 | 3 |

## O que aprendi

- **Passar o contrato foi o ajuste que mais pesou.** Sem ele, o modelo preencheu o vazio com o que é comum em APIs 5G: IMEI em vez de ICCID, rota `/activate`, PATCH de QoS, sessão PDU e 504 no timeout. Nada disso existe aqui.
- **Pedir "INVESTIGAR" em vez de deixar inventar** transformou lacunas do contrato em perguntas úteis. A melhor delas: se a compensação falhar, quem libera o SIM preso? O job de reconciliação não olha esse caso.
- **Dizer explicitamente "não gere teste de endpoint isolado"** foi o que fez os cenários serem de integração de verdade. Na versão simples, 10 de 12 casos eram de um endpoint só.
- **Exigir o mecanismo de simulação** (WireMock com atraso, k6 com barreira) acabou com casos vagos tipo "enviar requisições simultâneas" e deixou cada caso executável.
- **A lista "cobrir obrigatoriamente" manda mais que o objetivo.** O modelo cobre o que está enumerado. O Portal só apareceu nos casos porque estava na lista dos dois prompts refinados.
- A saída ainda precisa de revisão humana: valores de teste, códigos e expectativas foram conferidos contra o contrato.

## Estrutura

```
cenario/contrato-api.md     contrato dos 4 componentes
prompts/                    os 4 prompts (2 tipos × 2 versões)
casos-gerados/              saídas da LLM + revisão de cada uma
```
