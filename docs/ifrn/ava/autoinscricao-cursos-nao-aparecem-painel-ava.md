---
title: "Cursos com autoinscrição que não aparecem no Painel AVA"
category: "AVA"
service: "Painel AVA / Integrador / Moodle (tool_painelava) / SUAP"
audience: ["TIC", "Suporte", "Desenvolvedores"]
tags: ["autoinscrição", "painel-ava", "rule-engine", "vínculos", "suap", "moodle", "diagnóstico"]
reliability: "prático"
last_review: "2026-11-04"
source: "Investigação de casos reais de suporte; análise de código e histórico dos projetos"
---

## Sintoma

Um aluno não vê, na aba **Cursos com Autoinscrição** do Painel AVA, um ou mais cursos que deveria ver. Em geral ele vê outros cursos, mas não os que exige o seu curso ou modalidade. Às vezes o problema "volta" depois de ter sido dado como resolvido.

## Contexto

O Painel AVA decide se um curso aparece avaliando uma **regra** (sintaxe do [rule-engine](https://pypi.org/project/rule-engine/)) gravada no campo personalizado `restricoes_de_autoinscricao` do curso no Moodle. A regra é avaliada contra o **contexto do usuário**, montado pelo Painel:

- dados do perfil, como `tipo_usuario`;
- `outras_matriculas`: a lista de **vínculos** do usuário, cada um com `tipo`, `campus` e `detalhamento` (`modalidade`, `curso`, `nivel_ensino`, `ativo`).

Exemplo de regra típica:

```text
tipo_usuario in ['Aluno'] and $any([m['tipo'] == 'aluno' and m['detalhamento']['ativo'] == true
  and m['detalhamento']['modalidade'] in ['Técnico integrado ao ensino médio'] for m in outras_matriculas])
```

A comparação é **exata**, inclusive em maiúsculas. Qualquer diferença de texto entre o valor esperado na regra e o valor gravado no vínculo faz o curso sumir. E sumir **sem erro visível** para o aluno.

Já foram observadas quatro causas, que podem aparecer juntas:

| Causa | Como se manifesta |
|---|---|
| **Vínculos desatualizados no Painel** | Os vínculos não são regravados no login (ex.: a biblioteca de autenticação não lia a resposta **paginada** de vínculos do SUAP). Ficam com valores antigos. |
| **Valores renomeados na origem** | O SUAP renomeia um valor (ex.: a modalidade "Técnico Integrado" passa a "Técnico integrado ao ensino médio"). A regra usa um nome e o vínculo gravado, outro. |
| **Regra gerada por versão defeituosa do Integrador** | A regra usa campo que não existe no contexto (ex.: `situacao_diario`) ou listas vazias (`in []`). O curso não aparece para **ninguém**. |
| **Correção anterior parcial** | Um *fallback* aplicado "só quando não há vínculos gravados" não cobre quem já tinha vínculos antigos. Por isso o problema parece voltar. |

## Procedimento

### 1. Diagnosticar o usuário

Use a ferramenta **Diagnosticar** do Painel AVA: gaveta do usuário, só para superusuários. Na aba **Auto inscrição**, clique em **Ver regra** no curso. A regra aparece desenhada em blocos (verde = atendido, vermelho = não atendido), com o valor do usuário ao lado do esperado e cada vínculo detalhado.

Procure:

- comparação de `modalidade`/`curso` em vermelho com valores **parecidos, mas diferentes**: indica causa de nomenclatura ou vínculo desatualizado;
- erro `AttributeResolutionError: unknown attribute: '...'`: a regra usa um campo que não existe no contexto do Painel;
- comparação com lista vazia (`in []`): a regra foi gerada sem os valores.

### 2. Comparar vínculos gravados com os dados brutos

No admin do Painel, abra o usuário e compare:

- **Dados brutos**: a resposta do SUAP gravada no último login;
- **Vínculos**: o que o Painel usa nas regras.

Se os dados brutos têm o nome atual e os vínculos têm o antigo, ou se faltam vínculos, os vínculos estão **desatualizados**. A correção é na biblioteca de autenticação: regravar os vínculos no login, lendo todas as páginas da resposta. Depois do deploy, cada usuário é corrigido no **próximo login no Painel**. Login no Moodle não corrige. Para corrigir todos de uma vez, encerre as sessões no deploy (ex.: limpar o Redis/Valkey de sessões), para forçar novo login.

### 3. Encontrar regras problemáticas em cada Moodle

Consulta para *Configurable Reports*, sem dados de usuários. Ela classifica as regras e sugere a ação:

```sql
SELECT x.id, x.shortname, x.fullname, x.problema,
    CASE
        WHEN x.problema = '3. modalidade nova sem a antiga' THEN 'Nenhuma: corrige no login'
        WHEN COALESCE(x.diario_id, '') <> '' THEN 'Sincronizar no SUAP'
        ELSE 'Alterar no Moodle'
    END AS acao,
    x.regra
FROM (
    SELECT c.id, c.shortname, c.fullname,
        COALESCE(NULLIF(di.charvalue, ''), di.value) AS diario_id,
        COALESCE(NULLIF(d.charvalue, ''), d.value) AS regra,
        CASE
            WHEN COALESCE(NULLIF(d.charvalue, ''), d.value) LIKE '%situacao_diario%' THEN '1. situacao_diario'
            WHEN COALESCE(NULLIF(d.charvalue, ''), d.value) LIKE '%''Técnico Integrado''%'
              OR COALESCE(NULLIF(d.charvalue, ''), d.value) LIKE '%''Técnico Subsequente''%'
              OR COALESCE(NULLIF(d.charvalue, ''), d.value) LIKE '%''FIC''%' THEN '2. nome antigo de modalidade'
            WHEN COALESCE(NULLIF(d.charvalue, ''), d.value) LIKE '%Técnico integrado ao ensino médio%'
                THEN '3. modalidade nova sem a antiga'
        END AS problema
    FROM {course} c
    JOIN {customfield_data} d ON d.instanceid = c.id
    JOIN {customfield_field} f ON f.id = d.fieldid AND f.shortname = 'restricoes_de_autoinscricao'
    LEFT JOIN {customfield_field} fdi ON fdi.shortname = 'diario_id'
    LEFT JOIN {customfield_data} di ON di.fieldid = fdi.id AND di.instanceid = c.id
) x
WHERE x.problema IS NOT NULL
ORDER BY x.problema, acao, x.fullname
```

Amplie a lista de nomes antigos conforme as renomeações da origem. No PostgreSQL, o `LIKE` diferencia maiúsculas.

### 4. Corrigir as regras

- **Diário vindo do SUAP** (tem `diario_id`): sincronize de novo no SUAP. O Integrador regrava a regra com os valores atuais.
- **Sala de coordenação ou curso sem `diario_id`**: corrija a regra **no Moodle**. A sincronização **não atualiza** salas de coordenação já criadas.
- Durante uma renomeação, prefira regras que aceitem os **dois** nomes. Ex.: `in ['Técnico integrado ao ensino médio', 'Técnico Integrado']`. Não troque a regra só para o nome antigo: ela quebra quando os vínculos forem atualizados.

## Quando escalar

- Os dados brutos do SUAP vêm com valores inesperados, ou mudaram sem aviso: equipe do SUAP.
- Vínculos não são atualizados mesmo após novo login com a versão corrigida: desenvolvimento do Painel AVA e da biblioteca de autenticação.
- Muitas regras geradas com erro: desenvolvimento do Integrador.

## Equipe responsável

Equipe de desenvolvimento do Painel AVA e do Integrador, com apoio da equipe do SUAP para valores e formato da API.

## Links oficiais

- [rule-engine (sintaxe das regras)](https://zerosteiner.github.io/rule-engine/)
- [django-rule-engine (explicação em árvore e desenho em blocos)](https://django-by-kelsoncm.github.io/django-rule-engine/explicacao-em-blocos.html)
- [Integração SUAP e Moodle (visão geral)](integracao-suap-moodle-visao-geral.md)

## Nível de confiabilidade

- **Prático:** causas confirmadas na análise de casos reais e no histórico de código; consultas validadas em Moodle de produção.

## Notas adicionais

- Veja as lições de engenharia tiradas deste caso em [Lições de integração: APIs paginadas, valores renomeados e regras geradas](../../desenvolvimento/integracoes/licoes-integracao-apis-regras.md).
- Ao extrair dados de um usuário para reproduzir o caso em ambiente local ou de teste, leve só o necessário (perfil, vínculos, dados brutos) e remova dados pessoais (CPF, RG, e-mails, telefones, nascimento, filiação, fotos).
