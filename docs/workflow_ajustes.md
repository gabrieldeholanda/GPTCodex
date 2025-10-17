# Ajustes de Workflow para Busca de Imóveis

## Contexto

A análise dos logs dos atendimentos mostra dois comportamentos distintos:

1. **Execução correta** – após coletar cidade, dormitórios e orçamento, o agente chamou `BuscarImoveis`, recuperou 5 opções, enviou cada imóvel com `DetalhesImovel` + `sendImage` e finalizou com `assistant_message` adequado.
2. **Execução interrompida** – o agente reuniu todos os filtros (intenção *alugar*, cidade Hortolândia, 2 quartos, orçamento R$ 5.000), marcou `next_action":"buscar_imoveis"`, porém não disparou a ferramenta até receber a mensagem "ok" do usuário.

Esse atraso quebra a experiência: o lead precisa confirmar manualmente que deseja ver as opções, mesmo já tendo fornecido todos os critérios obrigatórios.

## Causa provável

O fluxo atual parece exigir uma confirmação textual antes de acionar `BuscarImoveis`. Como consequência, mesmo após `next_action":"buscar_imoveis"`, o agente envia apenas a frase "Vou buscar opções..." e aguarda um novo turno. Somente depois de uma resposta qualquer (ex.: "ok") ele executa a busca.

## Ajustes recomendados

### 1. Disparar ferramentas no mesmo turno

**Regra proposta:**

> Sempre que `next_action` assumir o valor `buscar_imoveis` e todos os campos obrigatórios (`intent`, `city`, `bedrooms`, `budget_max` ou equivalente) estiverem preenchidos, o orquestrador deve acionar `BuscarImoveis` imediatamente, no mesmo turno, antes de encerrar a resposta.

**Implementação sugerida:**

- No bloco de avaliação de próximos passos, inserir uma verificação antes de montar o `assistant_message` final:
  ```pseudo
  if next_action == "buscar_imoveis" and missing_fields vazio:
      chamar BuscarImoveis(qualification)
      processar retorno (Detalhes + sendImage)
      construir assistant_message final com os resultados
      atualizar next_action (p.ex. "perguntar" ou "agendar_visita")
  ```
- Remover qualquer guarda que dependa de "confirmação do usuário" após a coleta dos filtros.

### 2. Mensagens padrão pós-busca

Depois da busca automática, manter um `assistant_message` que esclareça que as opções já foram enviadas e proponha o próximo passo:

```
"assistant_message": "Acabei de te enviar por imagem as opções que encontrei em Hortolândia. Te interessa agendar uma visita a algum deles?"
```

Isso garante consistência com o caso em que a busca foi disparada apenas após o "ok".

### 3. Tratamento de respostas vazias ou não relacionadas

Para evitar regressões (ex.: usuário responde com outro tema enquanto a busca está sendo disparada), acrescentar uma salvaguarda:

- Se a mensagem do usuário for vazia ou irrelevante **antes** da coleta completa, manter `next_action":"perguntar"`.
- Assim que os dados obrigatórios estiverem presentes, a busca deve ocorrer independentemente do conteúdo textual da última mensagem.

### 4. Atualizar documentação do fluxo

- Revisar o diagrama/arquivo de workflow para registrar que a transição "Qualificação completa → BuscarImoveis" é automática.
- Destacar que o agente só deve aguardar confirmação quando houver ambiguidade nos filtros ou quando `missing_fields` não estiver vazio.

## Benefícios esperados

- Redução de atrito: o lead recebe as opções imediatamente.
- Menos turnos por atendimento e menor chance de abandono antes de ver os imóveis.
- Consistência com as regras obrigatórias de envio de imagens (evita esquecer de disparar o pacote de `DetalhesImovel` + `sendImage`).

## Próximos passos sugeridos

1. Implementar a alteração no orquestrador/estado do workflow conforme descrito.
2. Executar testes simulando a jornada de aluguel e compra para garantir que a busca ocorre no mesmo turno em ambos os intents.
3. Monitorar logs após o deploy para confirmar que a quantidade de turnos entre "tenho filtros completos" e "envio das imagens" caiu para 0.

