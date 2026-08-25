---
name: ironbug-padronizacao
description: Aplica o padrão de código da Ironbug — definido em references/padrao.md — a um projeto .NET com views Razor e AngularJS: nomenclatura em português no formato SalvadorDe/ListadorDe, camada de serviço obrigatória, projeções em vez de entidade inteira, AsNoTracking, async com CancellationToken, interpolação de string, tempo de vida correto na injeção de dependência, warnings resolvidos, e injeção em array inline no AngularJS. Use sempre que o usuário pedir "padronize o projeto", "deixe no padrão", "aplique o padrão da Ironbug", "esse código está fora do padrão", "uniformize a nomenclatura", ou quando ele apontar inconsistência de estilo entre arquivos, telas ou projetos — inclusive quando não usar a palavra "padrão". Entrega um relatório numerado do que está fora e aplica o que o usuário escolher.
---

# Padronização Ironbug

Aplica um padrão **já decidido**. A skill não escolhe convenção, não infere estilo do código
existente e não inventa regra: a fonte da verdade é `references/padrao.md`, e o trabalho é
medir a distância entre o projeto e aquele documento.

Isso é deliberado. O padrão nasceu de um mapa de divergência entre dois desenvolvedores, e a
decisão de qual estilo vence foi tomada por uma pessoa, uma vez. Uma skill que decidisse por
conta própria produziria um terceiro estilo que ninguém escreveu.

## Antes de qualquer coisa: leia o padrão

Leia `references/padrao.md` inteiro. São 15 pontos, e vários têm exceção explícita — aplicar
o título da regra sem a exceção causa dano. Três exemplos de como errar:

- `async` **não** é para todo método, só para os que fazem I/O.
- `AsNoTracking` **não** vai na consulta cujo resultado será alterado e salvo.
- "menos comentários" **não** é apagar todos: o que explica o porquê fica.

## Fase 1 — Medir a distância

Percorra os 15 pontos e conte, com caminho e linha. O que interessa é onde está, não quanto é.

Comandos que cobrem a maior parte:

```bash
# nomenclatura de serviço fora do padrão
grep -rn "class \w*\(Service\|Repository\|Manager\|Helper\)\b" --include="*.cs"

# regra de negócio ou consulta no controller (camada de serviço ausente)
grep -rn "_context\.\|DbContext" --include="*Controller.cs"

# leitura sem AsNoTracking / consulta síncrona com par async
grep -rn "\.ToList()\|\.FirstOrDefault()\|\.Any()\|\.Count()\|SaveChanges()" --include="*.cs"

# sync-over-async
grep -rn "\.Result\b\|\.Wait()\|GetAwaiter()\.GetResult()" --include="*.cs"

# CancellationToken: compare os dois números — o segundo deveria acompanhar o primeiro
grep -rc "async Task" --include="*.cs" . | awk -F: '{s+=$2} END {print "métodos async:", s}'
grep -rc "CancellationToken" --include="*.cs" . | awk -F: '{s+=$2} END {print "com token:", s}'

# consulta async que ignora o token que o método recebeu (o caso pior: aceita e não repassa)
grep -rn "Async()" --include="*.cs" | grep -v "SaveChangesAsync()"

# concatenação onde caberia interpolação
grep -rn '"\s*+\s*\w' --include="*.cs"

# regions
grep -rn "#region" --include="*.cs"

# AngularJS sem injeção em array (o de maior volume, e o que quebra minificado)
grep -rnE "\.(controller|service|factory|directive)\(\s*['\"][^'\"]+['\"]\s*,\s*function" --include="*.js"

# pastas js fora do par module.js + service.js
for d in */wwwroot/js/*/; do [ -f "$d/module.js" ] || echo "sem module.js: $d"; done
```

Para warnings, rode `dotnet build` e separe os do projeto dos que vêm de pacote.

Abreviação em nome de variável e mistura de idioma não saem por grep — precisam de leitura.
Amostre os arquivos mais tocados no `git log` em vez de tentar cobrir tudo.

## Fase 2 — Relatório

Numeração **global e contínua**, 1 a N, sem reiniciar por seção. O usuário responde por
número ("faça todos menos o 4", "faça o 2, 3 e 7"), e numeração repetida torna o pedido
ambíguo. O ponto do padrão vai como etiqueta na linha:

```
N. `[P11]` <o que está fora> — <quantas ocorrências, em quantos arquivos>
   Exemplos: `caminho/Arquivo.cs:linha`, `caminho/Outro.js:linha`
   Risco de mexer: <baixo|médio|alto> — <por quê, em uma linha>
```

Ordene por **risco de deixar como está**, não por volume nem por número do ponto. O que
quebra em produção vem primeiro; cosmético vem por último. Injeção AngularJS sem array, por
exemplo, é silenciosa até o dia em que alguém liga a minificação — e aí a tela inteira morre.

Feche dizendo quanto do projeto já está no padrão, ponto a ponto. É o número que responde
"falta muito?".

## Fase 3 — Aplicar

Aplique o que o usuário escolher, e todas as ocorrências do ponto escolhido — não a primeira.

Ordem que reduz retrabalho:

1. **Nomenclatura e estrutura primeiro** (pontos 8, 15). Renomear classe e extrair serviço
   mexe em assinatura; fazer depois obriga a refazer o que já foi ajustado.
2. **Depois o acesso a dados** (2, 3, 4, 16). Projeção, `AsNoTracking`, async e
   `CancellationToken` andam juntos no mesmo método — tocar uma vez só. O token em especial
   muda a assinatura de toda a cadeia, então fazê-lo junto com a conversão async evita
   assinar duas vezes o mesmo método.
3. **Depois o resto do C#** (1, 5, 6, 7, 9, 10, 12, 13, 14).
4. **AngularJS por último** (11), módulo a módulo. É o de maior volume e o mais mecânico.

Regras de segurança:

- **Renomeação é feita com verificação de consumidores.** Antes de renomear classe, método ou
  serviço, grep pelo nome antigo em todo o repositório — views `.cshtml`, registros de DI,
  rotas, arquivos `.js`. Nome que aparece em string não é pego pelo compilador.
- **Compile ao fim de cada grupo**, não ao fim de tudo. Um erro de renomeação no meio de 40
  arquivos custa caro para localizar.
- **AngularJS não tem compilador.** Depois de converter um módulo para injeção em array,
  abra a tela e confira o console. Erro de injeção só aparece em runtime.
- **Se a mudança altera tela, tire print** e mostre antes de dar por encerrado.
- Ponto do padrão que exige decisão nova — caso não previsto no `padrao.md` — **pare e
  pergunte**. Não decida em silêncio; a resposta vira uma linha nova no padrão.

## Armadilhas

- **Trocar `Include` por projeção sem olhar o consumidor.** Se a view usa uma propriedade que
  você não projetou, a tela quebra sem erro de compilação. Confira o `.cshtml`.
- **Marcar como `async` sem `await` dentro** para "cumprir o ponto 4". Piora e esconde.
- **Apagar comentário que registra decisão.** O ponto 1 pede menos comentário, não menos
  informação. Comentário que explica por que algo é singleton, por que uma consulta é separada
  ou por que uma ordem importa é justamente o que se mantém.
- **Extrair serviço arrastando o `DbContext` para dentro do controller.** O ponto 8 é sobre o
  controller **não** conhecer o banco; mover a consulta para uma classe nova e injetar o
  contexto no controller não resolve nada.
- **Converter os 195 registros AngularJS de uma vez.** Vá por módulo, com a tela aberta.
