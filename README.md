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
