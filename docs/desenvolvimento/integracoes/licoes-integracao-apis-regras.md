---
title: "Lições de integração: APIs paginadas, valores renomeados e regras geradas"
category: "Desenvolvimento"
service: "Integrações entre sistemas / rule-engine / Django"
audience: ["Desenvolvedores"]
tags: ["integração", "api", "paginação", "migração-de-dados", "rule-engine", "diagnóstico", "lições-aprendidas"]
reliability: "prático"
last_review: "2026-11-04"
source: "Investigação de um defeito de autoinscrição entre sistemas acadêmicos, Painel e Moodle"
---

## Sintoma

Uma funcionalidade que depende de dados vindos de outro sistema deixa de funcionar para parte dos usuários. Não há erro no log, a "correção" anterior não resolve todos os casos, e a primeira explicação ("o outro sistema mudou") parece certa, mas está incompleta.

## Contexto

Casos assim costumam ter **várias causas encadeadas**, cada uma inofensiva sozinha. No caso que originou estas notas:

1. o sistema de origem **paginava** a lista de vínculos (`{"count", "next", "results": [...]}`) e a biblioteca cliente esperava uma **lista simples**. Ela descartava a resposta sem erro, e os dados locais nunca mais foram atualizados;
2. uma **migração** de sistema copiou os dados locais tal como estavam e eles ficaram **congelados**;
3. o sistema de origem **renomeou valores textuais** (nomes de modalidade) que regras comparavam por igualdade exata;
4. regras salvas como texto tinham sido **geradas por uma versão defeituosa** de um gerador e só seriam regravadas numa nova sincronização;
5. uma correção anterior aplicou um *fallback* **só quando não havia dados**. Isso escondeu o problema para quem tinha dados antigos.

## Procedimento

### Lições para quem consome APIs

1. **Trate paginação explicitamente.** Se a resposta pode vir como `{"results": [...], "next": ...}`, siga o `next` até o fim, com um limite de páginas, e só no mesmo host. Nunca transforme silenciosamente um formato inesperado em lista vazia: registre um aviso.
2. **Não confie em textos de cadastro como chave.** Valores como nomes de modalidade, curso ou situação mudam. Prefira códigos ou identificadores. Se precisar comparar texto, aceite os nomes antigos e novos durante a transição.
3. **Verifique se o dado local é atualizado depois de uma migração.** Uma migração que copia dados deve ser seguida de um teste que prove que o fluxo normal, como o login, **reescreve** esses dados.
4. **Desconfie de *fallbacks* "só se estiver vazio".** Eles corrigem os casos novos e deixam os antigos com o mesmo defeito, o que dá a impressão de que o problema "voltou".

### Lições para quem gera regras ou configurações salvas como texto

1. **Valide na geração e ao salvar.** Uma regra que referencia um campo inexistente ou compara com uma lista vazia (`in []`) deve ser recusada, não gravada.
2. **Lembre que corrigir o gerador não corrige o que já foi gerado.** Tenha uma consulta que encontre os registros afetados e saiba como regravá-los, seja por nova sincronização ou por edição manual. Documente quais registros a sincronização **não** atualiza.

### Lições de investigação

1. **Teste a hipótese com a linha do tempo.** "O sistema de origem mudou" explicava o texto diferente, mas não por que o login não corrigia. Comparar o **dado bruto** (resposta atual da origem) com o **dado usado** (cópia local) mostrou o defeito local.
2. **Use o histórico do código como evidência.** `git log -S"<trecho>"` encontra quando um texto entrou ou saiu do código. `git tag --contains <commit>` mostra em que versão isso chegou à produção.
3. **Separe código de dado.** Uma renomeação feita num *fixture* só chega ao banco quando o deploy carrega esse *fixture*. A data do commit não é a data em produção.
4. **Escreva a nota técnica com o que foi verificado.** Marque como "não verificado" o que for dedução.

### Ferramenta: explicar a regra em vez de só dizer "não atende"

O que mais acelerou o diagnóstico foi **explicar a regra nó a nó** para um usuário específico. A biblioteca [django-rule-engine](https://pypi.org/project/django-rule-engine/) faz isso com `explain_tree()`, que monta a explicação a partir do AST do `rule-engine`, e a desenha em **blocos encaixados** (verde/vermelho), mostrando o valor do usuário ao lado do esperado e detalhando cada item de `$any`/`$all`:

```python
from django_rule_engine.explain_tree import explain_tree
from django_rule_engine.blocks import render_blocks

explicacao = explain_tree(regra, contexto)   # árvore serializável em JSON
html = render_blocks(explicacao)             # ou RuleBlocks.render(el, explicacao) no navegador
```

Boas práticas para uma ferramenta desse tipo:

- restringir o acesso a administradores, no front **e** no back;
- mascarar campos sensíveis (CPF, e-mail, telefone...), inclusive dentro de dicionários aninhados;
- não colocar dados pessoais em eventos de analytics: só categorias e contagens;
- mostrar a tabela-resumo e abrir o detalhe sob demanda (ex.: modal grande), para não poluir a listagem.

## Quando escalar

Quando a causa estiver no sistema de origem (formato da API ou valores de cadastro), combine com a equipe dele um **aviso prévio** de mudanças em valores e formatos consumidos por integrações.

## Equipe responsável

Desenvolvedores dos sistemas integrados.

## Links oficiais

- [rule-engine](https://zerosteiner.github.io/rule-engine/)
- [django-rule-engine: explicação em árvore e desenho em blocos](https://django-by-kelsoncm.github.io/django-rule-engine/explicacao-em-blocos.html)

## Nível de confiabilidade

- **Prático:** lições tiradas de um caso real, com causas confirmadas no código e no histórico dos projetos.

## Notas adicionais

O caso concreto, com o diagnóstico e as consultas, está em [Cursos com autoinscrição que não aparecem no Painel AVA](../../ifrn/ava/autoinscricao-cursos-nao-aparecem-painel-ava.md).
