---
name: ironbug-varredura
description: Auditoria sistemática de um projeto inteiro (não do diff) em busca de bugs, falhas de regra de negócio, gargalos de performance, duplicação e desvios de padrão — entrega um relatório numerado para o usuário escolher o que aplicar, e então aplica em lote. Use sempre que o usuário pedir "faça uma varredura", "analise o projeto", "procure melhorias de performance", "revise o código buscando falhas", "busque padronizações e melhorias", "o que dá para melhorar aqui?" ou qualquer auditoria ampla — inclusive quando ele não usar a palavra "varredura". Para revisar só as mudanças pendentes de um commit ou PR, use /code-review; esta skill é para o projeto todo.
---

# Varredura

Auditoria ampla de um projeto: encontrar o que está errado ou mal resolvido no código
que já existe, apresentar em lista numerada, e aplicar o que o usuário aprovar.

A diferença para `/code-review` importa: aquele revisa um diff, esta revisa a base
inteira. Se o usuário está prestes a commitar algo, `/code-review` é mais barato e mais
preciso. Esta skill é para quando ele quer saber o estado geral do projeto.

## O que faz esta varredura valer alguma coisa

Um relatório de 40 itens genéricos ("considere extrair este método", "adicione tratamento
de erro") é pior que nenhum relatório: o usuário lê, não confia, e ignora. O que dá valor
é **cada item ter um cenário concreto de falha** — entradas ou estado específicos que
produzem comportamento errado, lentidão mensurável ou retrabalho real.

Quando você não consegue descrever como algo quebra, isso é sinal de que falta
investigação, não de que o achado é fraco. Volte ao código e descubra o cenário. Só
descarte depois de verificar que está realmente correto — descartar por preguiça de
investigar é como uma varredura perde justamente os achados que importam.

## Fase 1 — Delimitar o escopo real

Antes de ler qualquer código, descubra o que é código do projeto. Repositórios .NET e
Flutter carregam muito arquivo que não é seu e vai poluir tudo:

```bash
git ls-files | grep -vE 'wwwroot/lib/|node_modules/|/dist/|\.min\.|\.map$|/obj/|/bin/|\.g\.dart$|\.freezed\.dart$|Migrations/' > /tmp/escopo.txt
wc -l /tmp/escopo.txt
```

Confira o tamanho antes de continuar. Menos de ~300 arquivos: leia direto. Acima disso,
use a Fase 2 por dimensão para não estourar contexto lendo tudo linearmente.

Leia também o `CLAUDE.md` do projeto se existir — ele diz as convenções que valem ali, e
divergir delas é um achado legítimo. Se não existir, vale sugerir `/init` ao final.

## Fase 2 — Varrer por dimensão, não por arquivo

Varrer arquivo a arquivo faz você perder o padrão: o mesmo erro repetido em 12 telas
aparece como 12 achados desconexos. Varrer por dimensão faz o padrão emergir.

As dimensões que valem a pena são ortogonais entre si — cada uma enxerga o que as outras
não enxergam:

1. **Regra de negócio** — fluxos com estado (aprovação, pagamento, assinatura, sinistro):
   transições impossíveis, ausência de validação em uma das entradas do fluxo, condição
   invertida, permissão checada numa rota e esquecida na irmã.
2. **Performance de backend** — N+1, consulta dentro de laço, `Include` em excesso,
   método síncrono em caminho async, materialização de coleção inteira para tirar um
   `Count`, ausência de paginação.
3. **Performance/robustez de frontend** — nas views: chamadas em série que poderiam ser
   paralelas, listagem sem paginação, cálculo pesado no template.
4. **Duplicação** — trecho repetido 2+ vezes que pede helper. Esta é a que o usuário mais
   pede explicitamente; leve a sério e mostre todas as ocorrências, não só uma.
5. **Padrão e consistência** — divergência das convenções do próprio projeto: nomes,
   estrutura de pastas, jeito de tratar erro, jeito de montar tela.
6. **Segurança e dependências** — segredo versionado, endpoint sem autorização, e o
   estado dos pacotes (comandos abaixo).

Se o projeto for grande e o usuário tiver pedido paralelismo, cada dimensão vira um
agente independente e você consolida no final removendo duplicatas. Sem esse pedido,
faça em série — é mais lento, mas não gasta o orçamento do usuário sem ele saber.

### Comandos que valem por dezenas de greps

```bash
# .NET — pacotes vulneráveis e desatualizados (rápido e sempre rende achado real)
dotnet list package --vulnerable --include-transitive
dotnet list package --outdated

# Flutter/Dart — análise estática do próprio SDK
flutter analyze
flutter pub outdated
```

`flutter analyze` já pega `BuildContext` usado depois de `await`, `async` sem `await` e
dezenas de outros. Rode antes de procurar isso na mão.

## Fase 3 — Filtrar antes de reportar

Esta fase é a que separa uma varredura útil de uma lista de ruído. Antes de escrever cada
item, verifique:

- **Está mesmo errado?** Abra o código ao redor, a definição do tipo, o registro no DI.
  Muita coisa que parece bug some quando você lê a classe vizinha — por exemplo, um
  filtro de exclusão lógica "faltando" numa consulta e presente na outra costuma ser
  correto porque só uma das entidades tem exclusão lógica.
- **Já foi resolvido?** `git log -S"<trecho>"` e `git blame` mostram se aquilo é dívida
  viva ou coisa que a próxima linha já trata.
- **Vale o custo de mexer?** Um achado que exige refatorar 30 arquivos para ganhar 2ms
  não é um achado, é uma armadilha.

O que fazer com o resto importa tanto quanto o filtro. **Consolide, não descarte.** Doze
telas com o mesmo defeito são um item que lista as doze ocorrências, não doze itens nem
um item que cita só a primeira. Achados diferentes com a mesma raiz (por exemplo, seis
endpoints sem recorte de empresa) também viram um item — a correção é uma só.

Achado pequeno mas real continua entrando, no fim da lista e ainda numerado. O único
destino do descarte é a seção "o que eu conferi e estava certo": você olhou, não é
problema, e explica por quê. Se um item saiu da lista sem passar por essa seção, ele foi
perdido, não filtrado.

Antes de fechar o relatório, faça uma passada perguntando: *o que eu vi e não escrevi?*
Segurança e regra de negócio são as primeiras vítimas de um filtro apertado demais,
porque o cenário de falha delas exige entender o domínio — e é justamente onde o usuário
mais precisa de você.

## Fase 4 — O relatório

O usuário responde a este relatório escolhendo itens pelo número: "faça todos", "faça
todos menos o 3", "pode remover o 2 e o 3", "faça o 2, 3, 7". É assim que ele trabalha, e
é por isso que **a numeração é global e contínua** — 1 a N do começo ao fim, sem reiniciar
em nenhuma seção. Havendo dois itens "3" no relatório, "faça todos menos o 3" fica
ambíguo e o usuário precisa reescrever o pedido inteiro. Uma única lista corrida é a
diferença entre ele responder com quatro palavras ou ter que redigir um parágrafo.

Por isso também **não agrupe os itens em seções por categoria** (regra de negócio,
performance, padronização). A tentação de agrupar é forte porque organiza o texto, mas
ela sempre acaba reiniciando a contagem. Quando a categoria for útil, marque-a **dentro
da linha do item**, como etiqueta — `[Regra]`, `[Perf]`, `[Segurança]`, `[Duplicação]` —
mantendo uma lista só. Você informa a categoria e preserva a numeração.

Ordene por gravidade real: o que quebra em produção ou perde dado primeiro, o que só
custa desempenho depois, o que é higiene por último.

Formato de cada item:

```
N. `[Etiqueta]` <o que está errado, em uma linha> — `caminho/do/arquivo.cs:linha`
   Quebra quando: <cenário concreto: entrada, estado ou volume que produz o erro>
   Correção: <o que fazer, em uma linha — não o código inteiro>
```

Quando o item cobrir várias ocorrências, liste todos os arquivos e linhas afetados — é o
que permite ao usuário dimensionar o trabalho antes de escolher.

Feche o relatório com uma linha só: quantos itens, e quais você faria primeiro se fosse
sua a decisão. O usuário decide, mas quer sua opinião.

**Diga também o que você verificou e estava certo** — dois ou três pontos que pareciam
suspeitos e não eram. Isso poupa o usuário de investigar o mesmo depois, e mostra onde
você olhou de verdade.

## Fase 5 — Aplicar

Quando o usuário escolher os itens, aplique todos os escolhidos na mesma sessão — ele
prefere o lote resolvido a uma investigação item a item.

Ao aplicar:

- Corrija **todas** as ocorrências do padrão, não a primeira. Se o item 4 dizia "12 telas
  usam a classe obsoleta", as 12 mudam.
- Rode o build ao terminar (`dotnet build`, `flutter analyze`) e mostre o resultado. Uma
  varredura que deixa o projeto sem compilar é pior que nenhuma.
- Se a mudança altera tela, tire print e mostre antes de dar por encerrado.
- Se um item escolhido se revelar mais complexo do que o relatório sugeria, pare nele,
  diga por quê, e siga com os outros. Não o resolva pela metade em silêncio.

## Armadilhas conhecidas

- **Contar arquivos alterados no `git status` como se fossem código.** Em projeto .NET
  MVC, dezenas de "arquivos alterados" costumam ser bootstrap e jquery versionados.
  Filtre antes de tirar qualquer conclusão sobre o tamanho do trabalho.
- **Reportar o que o projeto já faz bem.** Se a base já usa `AsNoTracking`,
  `CancellationToken` e consultas agregadas, não gere itens dizendo para adicioná-los.
  Verifique antes de recomendar.
- **Confundir estilo com defeito.** Comentário em excesso, nome fora do padrão e método
  longo são itens legítimos de padronização, mas entram abaixo dos que quebram —
  e só se o usuário tiver pedido esse eixo.
- **Sugerir migração de framework.** A varredura melhora o que existe. Trocar ORM,
  atualizar major version ou reescrever camada é outra conversa, e deve ser oferecida
  como observação no final, nunca como item numerado.
