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

### `/ironbug-padronizacao`

Aplica o padrão de código da Ironbug a um projeto .NET com Razor e AngularJS. A skill **não
escolhe** convenção: a fonte da verdade é [`skills/ironbug-padronizacao/references/padrao.md`](skills/ironbug-padronizacao/references/padrao.md),
e caso não previsto ali faz a skill parar e perguntar em vez de decidir sozinha.

Dispara com *"padronize o projeto"*, *"deixe no padrão"*, *"esse código está fora do padrão"*,
*"uniformize a nomenclatura"*.

**Os 15 pontos** cobrem nomenclatura em português no formato `SalvadorDe`/`ListadorDe`, camada
de serviço obrigatória, projeções em vez de entidade inteira, `AsNoTracking`, `async` com
`CancellationToken`, interpolação de string, tempo de vida correto na injeção de dependência,
warnings resolvidos, e o par `module.js` + `service.js` com injeção em array inline no
AngularJS.

**Vários pontos têm exceção explícita** e elas fazem parte da regra: `async` só onde há I/O,
`AsNoTracking` exceto quando a entidade será alterada, e "menos comentários" preserva os que
explicam o porquê.

**Como o padrão foi definido:** a partir de um mapa de divergência entre dois desenvolvedores
no SendCase (13.187 contra 3.244 linhas com dono claro), medindo onde os estilos realmente
diferiam. As decisões foram tomadas uma vez, por uma pessoa, e escritas — para que a
ferramenta não produza um terceiro estilo que ninguém escreveu.

**Validada no Line**, projeto que não originou o padrão: 22 itens. O relatório separou os 12
registros AngularJS que entram no bundle minificado (quebram a tela em runtime) dos 20 que
ficam fora — mesmo ponto do padrão, riscos diferentes.
