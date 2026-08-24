---
name: ironbug-lazyloading-async
description: Mapeia o que falta para remover o lazy loading do EF Core e converter a aplicação para async de ponta a ponta — combina inventário estático com instrumentação em runtime e entrega um plano faseado A/B/C, separando bloqueadores de ganho de escala. Use sempre que o usuário pedir "mapeie o que falta para tirar o lazy loading", "quero deixar tudo async", "remover os métodos síncronos", "o que falta para chegar nisso", "quanto falta da migração async", ou quando estiver planejando sair de lazy loading / sync-over-async num projeto .NET — inclusive quando ele descrever o sintoma (N+1, consulta dentro de laço, thread starvation, requisição lenta sob carga) sem nomear a causa. Entrega um mapa e um plano, não a refatoração inteira de uma vez.
---

# Mapa de saída do lazy loading e do código síncrono

Duas migrações que caminham juntas em projeto .NET legado: tirar o lazy loading do EF Core
e converter a camada de dados para async. Andam juntas porque a mesma navegação preguiçosa
dentro de um laço é, ao mesmo tempo, um N+1 e um ponto onde alguém vai enfiar `.Result`
para "resolver" o async.

O que esta skill entrega é **um mapa com plano faseado**, não a refatoração completa. Em
base legada, a migração inteira de uma vez não é revisável nem tem rollback parcial — e o
usuário precisa escolher por onde começar antes de qualquer linha mudar.

## Por que o inventário estático não basta

Procurar `virtual` nas entidades diz o que **pode** carregar preguiçosamente. Não diz o que
**de fato** carrega, nem quantas vezes, nem em qual tela. Um projeto com 200 navegações
virtuais pode ter 15 que realmente disparam em produção — e são essas 15 que importam.

Por isso o mapa tem duas metades, e a segunda é a que dá confiança:

## Fase 1 — Inventário estático

O que procurar, e o que cada achado significa:

```bash
# navegações que podem carregar preguiçosamente
grep -rn "public virtual " --include="*.cs" | grep -vE "virtual (string|int|bool|decimal|DateTime|Guid)"

# o lazy loading está ligado? por proxy ou por injeção?
grep -rn "UseLazyLoadingProxies\|ILazyLoader\|LazyLoadingEnabled" --include="*.cs"

# sync-over-async: cada ocorrência num caminho de request é candidata a bloqueador
grep -rn "\.Result\b\|\.Wait()\|GetAwaiter()\.GetResult()" --include="*.cs"

# consultas síncronas que têm par async
grep -rn "\.ToList()\|\.FirstOrDefault()\|\.Any()\|\.Count()\|SaveChanges()" --include="*.cs"

# controllers e actions que ainda não são async
grep -rn "public IActionResult \|public ActionResult " --include="*.cs"
```

Registre onde cada coisa está, não só quantas são. A contagem sozinha não permite fasear.

## Fase 2 — Inventário em runtime

Esta é a parte que distingue um mapa útil de uma lista de suspeitos. O EF Core avisa toda
vez que uma navegação carrega preguiçosamente, e dá para capturar isso:

- Assine o evento de lazy loading do `DbContext` (`NavigatedLazyLoading` traz `entityType` e
  `navigation`).
- Remova o sufixo `Proxy` do `entityType` — sem isso os nomes não batem com as classes do
  código e o inventário fica inútil para localizar o ponto.
- Deixe **desligado por padrão**, atrás de config. A instrumentação precisa custar zero
  quando não está em uso, senão ninguém deixa ligado o tempo suficiente para colher dado.
- Exercite os fluxos principais com ela ligada — as telas mais acessadas, e principalmente
  os caminhos de escrita.

O resultado é uma lista de (entidade, navegação, tela, quantas vezes) que diz onde o
problema realmente vive. Sem essa lista, o plano vira chute ordenado por tamanho de arquivo.

## Fase 3 — Classificar em A, B e C

Agrupar por complexidade é o que torna a migração executável em pedaços com rollback:

- **C — bloqueadores.** Precisam sair primeiro porque impedem o resto ou quebram em
  produção: navegação preguiçosa dentro de laço, navegação disparada durante serialização
  (a resposta da API materializa meio banco), `.Result` em caminho de request, e qualquer
  lazy load em fluxo de escrita crítica.
- **A — ganho de escala.** Correção mecânica repetida em muitos lugares: trocar navegação
  preguiçosa por projeção ou `Include` explícito, converter consulta síncrona no par async.
  Baixo risco, alto volume — é onde a barra de progresso anda.
- **B — otimização pontual.** Ganho pequeno, escopo isolado. Fica por último e pode nunca
  ser feito sem prejuízo.

Ataque **C primeiro**, depois **A**, e adie **B**. É contra-intuitivo começar pelo mais
difícil, mas C é o que bloqueia: enquanto existir `.Result` no meio do caminho, converter
o resto para async não entrega o ganho de throughput que justifica a migração.

## Fase 4 — O relatório

Numeração **global e contínua**, 1 a N, sem reiniciar por grupo — o usuário responde por
número ("faça os C", "faça o 2, 3 e 7"), e numeração repetida torna o pedido ambíguo. A
fase aparece como etiqueta dentro da linha, não como seção separada:

```
N. `[C]` <o que está errado> — `caminho/Arquivo.cs:linha`
   Dispara em: <tela/fluxo, e quantas vezes no runtime instrumentado>
   Custa: <o efeito concreto: N+1 de 340 consultas, thread bloqueada, resposta de 4s>
   Correção: <projeção, Include explícito, conversão async — em uma linha>
```

Feche com três números que o usuário quer saber sem perguntar: quantas navegações
preguiçosas ainda disparam, quantos pontos sync-over-async restam, e quanto do caminho de
dados já está async. É isso que responde "quanto falta".

## Fase 5 — Aplicar com segurança

Quando o usuário escolher os itens:

- **`ToListAsync()` antes do `foreach`.** Iterar um `IQueryable` mantém o DataReader aberto
  enquanto o corpo do laço executa async — materialize antes. Vale especialmente quando há
  `SaveChanges` dentro do laço.
- **Suba o `Task` até o controller.** Se o método faz I/O ou enfileira job, ele é async, e
  a assinatura sobe pela cadeia inteira. Nunca use `.Result` para "ligar" um trecho async a
  um chamador síncrono — isso recria o bloqueador que você acabou de remover.
- **Propague o `CancellationToken`.** Converter para async sem token entrega metade do
  benefício: a requisição abandonada continua consumindo banco.
- **Prefira projeção a `Include`.** Trocar lazy loading por `Include` em coleções irmãs
  gera produto cartesiano — às vezes pior que o N+1 original. Se a tela só mostra três
  campos, projete três campos.
- **Teste por camada de risco: escrita crítica antes de leitura.** Uma leitura errada mostra
  número errado; uma escrita errada perde dado.
- **Valide visualmente todo número que mudou de fonte.** Quando um total sai de coleção em
  memória para agregação no banco, o valor pode mudar de verdade (itens excluídos logicamente,
  duplicatas do join). Compare a tela antes e depois.
- **Harness isolado quando o comportamento do EF for a dúvida.** SQLite in-memory com
  proxies de lazy loading reproduz o runtime real sem depender de banco externo, e responde
  em segundos o que levaria um deploy para descobrir.

## Armadilhas conhecidas

- **Desligar `UseLazyLoadingProxies` de uma vez.** Tudo que dependia de carga implícita passa
  a vir nulo, silenciosamente, e o erro aparece longe da causa. Remova as dependências
  primeiro; o desligamento global é o **último** passo, não o primeiro.
- **`virtual` que não é do EF.** Frameworks de mock e algumas bibliotecas exigem `virtual`.
  Antes de remover o modificador, confira quem depende dele.
- **`async` sem `await`.** Marcar o método como async para "padronizar" sem tornar a chamada
  interna assíncrona não muda nada e ainda esconde o problema do próximo inventário.
- **Confundir método síncrono com bloqueador.** Código síncrono que não faz I/O (cálculo,
  formatação, mapeamento) deve continuar síncrono — convertê-lo só adiciona máquina de estado.
- **Contar `virtual` como progresso.** A métrica que importa é quantas navegações **disparam**
  no runtime instrumentado, não quantas existem no código.
