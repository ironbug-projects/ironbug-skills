# Ironbug Skills

Catálogo de skills da Ironbug para o Claude Code.

## Instalação

```bash
claude plugin marketplace add ironbug-projects/ironbug-skills
```

```bash
claude plugin install ironbug-skills
```

Para receber atualizações depois:

```bash
claude plugin marketplace update ironbug-skills
```

## Skills

### `/ironbug-varredura`

Auditoria sistemática de um projeto inteiro — não do diff. Procura bugs, falhas de regra
de negócio, gargalos de performance, duplicação e desvios de padrão, entrega um relatório
numerado para escolher o que aplicar, e então aplica em lote.

Dispara com pedidos como *"faça uma varredura no projeto"*, *"procure melhorias de
performance"*, *"analise o projeto buscando falhas"* — inclusive sem a palavra "varredura".

Para revisar apenas as mudanças pendentes de um commit ou PR, use `/code-review`; esta
skill é para o projeto todo.

**O que ela garante no relatório:**

- Numeração global e contínua (1 a N), para responder por número: "faça todos menos o 3"
- Cada item com arquivo, linha e um cenário concreto de falha — não recomendação genérica
- Ordenação por gravidade, com etiqueta de categoria inline (`[Regra]`, `[Perf]`, `[Segurança]`)
- Uma seção do que foi verificado e estava correto, para não reinvestigar depois

**Medido em três projetos reais** (dois .NET MVC e um app Flutter), comparando execuções
com e sem a skill: relatórios de 5 a 8 vezes menores, com cenário de falha em 100% dos
itens contra praticamente nenhum na execução sem ela.

### `/ironbug-lazyloading-async`

Mapeia o que falta para remover o lazy loading do EF Core e converter a aplicação para
async de ponta a ponta. Entrega um mapa com plano faseado — não a refatoração inteira de
uma vez, que em base legada não é revisável nem tem rollback parcial.

Dispara com *"mapeie o que falta para tirar o lazy loading"*, *"quero deixar tudo async"*,
*"remover os métodos síncronos"* — e também quando o pedido descreve só o sintoma (N+1,
consulta dentro de laço, requisição lenta sob carga).

**O que a distingue:** o inventário estático diz o que *pode* carregar preguiçosamente, não
o que *carrega*. A skill combina o grep com instrumentação do evento `NavigatedLazyLoading`
do EF, desligável por config e com custo zero quando desligada.

Classifica em três grupos — **C** bloqueadores, **A** ganho de escala, **B** otimização — e
manda atacar C primeiro: enquanto houver `.Result` no caminho, converter o resto para async
não entrega o ganho que justifica a migração.

Fecha respondendo "quanto falta" com três números: navegações que ainda disparam, pontos de
sync-over-async restantes, e quanto do caminho de dados já é async.

**Validada no SendCase** (6 projetos com proxies ligados, 198 navegações virtuais): 24 itens
mapeados, 12 deles bloqueadores. O achado que justifica a Fase 2 apareceu no próprio teste —
18 dos 21 candidatos a N+1 encontráveis por grep já tinham `Include`, ou seja, o resíduo real
é invisível à análise estática.
