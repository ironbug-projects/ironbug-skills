# Padrão de código Ironbug

Decidido por Vinicius em 2026-08-25, a partir do mapa de divergência entre o código dele e
o do Erick no SendCase (13.187 contra 3.244 linhas, arquivos com dono claro).

Este arquivo é a **fonte da verdade**. A skill aplica o que está aqui e não inventa regra
nova: se um caso não está previsto, ela pergunta em vez de decidir sozinha.

---

## 1. Comentários — menos

Remova o comentário que apenas repete o que o código já diz (`// salva o usuário` sobre um
`SalvarUsuario()`). Mantenha o que explica **por quê**, o que registra decisão não óbvia, e
o que avisa de armadilha.

Modelo do que fica, tirado do próprio código: `SendCase.Domain/Cids/ListadorDeCids.cs`
documenta por que a classe é singleton e o que a implementação anterior fazia de errado.
Isso não é ruído — é o que impede alguém de "consertar" o registro no futuro.

Na dúvida: se apagar o comentário faz perder informação que não está no código, ele fica.

## 2. Projeções — mais

Prefira `.Select(x => new XxxViewModel { ... })` a materializar a entidade inteira. Traga do
banco só as colunas que a tela usa.

Isso reduz o tráfego, elimina o produto cartesiano que o `Include` de coleções irmãs gera, e
corta pela raiz a carga preguiçosa — a navegação que não foi projetada não existe para o EF.

`Include` continua válido quando a entidade completa é realmente necessária (edição, regra de
domínio que percorre a navegação).

## 3. `AsNoTracking` sempre que possível

Toda consulta de leitura leva `AsNoTracking()`. A exceção é quando a entidade vai ser
alterada e salva na mesma unidade de trabalho — aí o rastreamento é o mecanismo.

## 4. `async` sempre que possível

Todo método que faz I/O — banco, HTTP, arquivo, fila — é `async Task` e usa o par assíncrono
(`ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync`, `AnyAsync`, `CountAsync`). A
assinatura sobe pela cadeia até o controller.

Duas regras que evitam o efeito contrário:

- **Nunca** `.Result`, `.Wait()` ou `.GetAwaiter().GetResult()` para ligar código async a um
  chamador síncrono. Converta o chamador.
- Código que **não** faz I/O (cálculo, formatação, mapeamento) continua síncrono. Marcar como
  `async` sem `await` só adiciona máquina de estado e esconde o problema.

Propague `CancellationToken` até a consulta. Sem ele, a requisição abandonada continua
consumindo banco.

## 5. Interpolação de string

`$"Olá {nome}"` em vez de `"Olá " + nome`. Vale também para montagem de mensagem, log e
caminho. É o único ponto do padrão em que o estilo do Erick prevaleceu — ele já usava quase
o dobro.

## 6. Sem `#region`

Nenhum `#region` no código novo, e remova os existentes ao tocar no arquivo. Região que
esconde um bloco grande é sinal de que a classe deveria ser dividida — resolva a causa.

## 7. `try/catch` — com propósito

Use `try/catch` quando ele **acrescenta** alguma coisa:

- falha esperada e tratável: integração externa fora do ar, parse de arquivo do usuário,
  arquivo ausente;
- contexto que o erro cru não tem: qual registro, qual integração, qual etapa;
- limpeza obrigatória, quando `using` não resolve.

Não use para capturar, logar e relançar sem acrescentar nada — os projetos já têm tratamento
global de exceção no `Program.cs`, e o `catch` vazio de valor só afasta o erro da causa.

Nunca engula exceção silenciosamente. `catch` sem log e sem tratamento é proibido.

## 8. Sempre camada de serviço

Controller não contém regra de negócio nem consulta ao banco. Ele recebe, delega e devolve.

A regra vive numa classe de serviço no `.Domain`, com interface, registrada no módulo de
injeção. O controller depende da interface, nunca da implementação nem do `DbContext`.

## 9. Tempo de vida correto nas injeções

- **Scoped** é o padrão. Tudo que depende de `DbContext` é scoped, porque o `DbContext` é
  scoped.
- **Singleton** só para estado imutável e thread-safe, sem nenhuma dependência scoped.
  Exemplo correto no código: `IListadorDeCids` — catálogo CID-10 carregado de recurso
  embutido, imutável, sem `DbContext`.
- **Transient** para objeto barato e sem estado. Se o serviço guarda estado por requisição,
  ele é scoped, não transient.

O erro a evitar é a dependência cativa: singleton que recebe um serviço scoped mantém o
primeiro `DbContext` vivo para sempre. Ao registrar um singleton, confira o construtor.

## 10. Warnings resolvidos

O build não introduz warning novo. Ao tocar num arquivo, resolva os warnings dele — sobretudo
os de nulabilidade, `await` faltando e membro obsoleto.

Warning que a equipe decidiu aceitar leva supressão explícita com justificativa no código.
Suprimir sem justificativa é o mesmo que ignorar.

## 11. AngularJS: `module.js` + `service.js` e injeção em array

Cada funcionalidade tem sua pasta em `wwwroot/js/<Funcionalidade>/` com:

- `module.js` — controllers e diretivas da funcionalidade;
- `service.js` — acesso à API, nada de manipulação de DOM.

**Toda** injeção usa a forma de array inline, que sobrevive à minificação:

```js
// certo
app.controller('SalvarUsuarioController', ['$scope', '$usuarios', function ($scope, $usuarios) {
    // ...
}]);

// errado — o minificador renomeia $scope e a injeção quebra em produção
app.controller('SalvarUsuarioController', function ($scope, $usuarios) {
    // ...
});
```

Este é o ponto de maior volume do padrão: no SendCase são **195 registros na forma errada
contra 11 na forma certa**, espalhados por 42 módulos.

## 12. Sem abreviação em nome de variável

`quantidadeDeItens`, não `qtdItens`. `usuario`, não `usr`. `configuracao`, não `cfg`.

Aceitos por convenção universal: `id`, `i`/`j` em laço curto, e siglas de domínio que a
equipe usa falando (`cid`, `cpf`, `cnpj`, `ubs`).

## 13. Métodos em português, sem abreviar

Nome de método, classe e propriedade em pt-BR por extenso: `SalvarAtendimento`,
`ListarEspecialidades`, `ValidarCertificadoDigital`.

Não misture idiomas no mesmo nome (`SaveAtendimento`, `ListarSpecialties`). Termos técnicos
sem tradução corrente permanecem em inglês (`Token`, `Cache`, `Middleware`, `Endpoint`).

Sufixo `Async` no nome do método assíncrono — é a única exceção à regra do português, e já é
o que os dois lados faziam.

## 14. SOLID sem exagero

O que se cobra:

- Uma responsabilidade por classe. Serviço que salva não lista.
- Depender de interface, não de implementação — que é exatamente o que a camada de serviço
  do item 8 já entrega.

O que **não** se cobra:

- Interface para tudo. A interface do serviço é a convenção; não crie mais camadas de
  abstração em cima dela.
- Quebrar classe em micro-classes para "cumprir" o princípio.
- Padrão de projeto onde um método resolve.

Regra prática: se a abstração não tem uma segunda implementação real nem um teste que a
substitua, ela provavelmente não deveria existir.

## 15. Nomenclatura dos serviços: `AcaoDeSubstantivo`

Classe de serviço nomeada pela ação que executa:

`SalvadorDeAtendimento`, `ListadorDeEspecialidades`, `SelecionadorDeRegulador`,
`RemovedorDeExameFavorito`, `ValidadorDeCertificadoDigital`, `CopiadorDeModelo`,
`MontadorDeRelatorio`, `EnviadorDeEmail`.

A interface é o mesmo nome com `I` na frente: `ISalvadorDeAtendimento`.

**Não** use os sufixos `Service`, `Repository`, `Manager` ou `Helper`. Este é o ponto em que
o código atual do próprio Vinicius é inconsistente — ele usa as duas convenções, e o Erick
não mistura. A regra unifica na forma em português.

---

## O que já era consenso — não é mudança, é registro

Medido nos dois lados e mantido:

- `IActionResult` como retorno de action (`ActionResult<T>` não é usado por ninguém);
- os helpers `Success()` / `Error()` das controllers base;
- sufixo `Async` no nome de método assíncrono;
- `var` para inferência quando o tipo é evidente na linha;
- `Include` quando a entidade completa é necessária.

## Ponto deliberadamente em aberto

`if` sem chaves aparece nos dois lados (31,8 contra 22,5 por mil linhas). Não há divergência
a resolver, e nenhuma decisão foi tomada. A skill **não** deve reescrever isso — se virar
regra um dia, entra aqui primeiro.
