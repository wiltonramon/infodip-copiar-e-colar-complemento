# Botão Copiar e Colar complemento no INFODIP

Script de usuário para o **INFODIP** (TSE) que preenche o campo **Complemento** com o número da comunicação, corrige repetições e copia o texto final para a área de transferência com um único clique.

- **Versão:** 4.6.2
- **Autor:** Ramon Machado
- **Licença:** GPL-3.0-or-later
- **Requer:** Tampermonkey (ou gerenciador compatível com `GM.setClipboard`)

---

## O que o script faz

O script procura o campo **Complemento** e o número da comunicação (formato `número/ano-UF`, por exemplo `269/2026-CE`) no mesmo modal visível. Quando identifica um único número, coloca o botão **Copiar e Colar** ao lado dele. Ao clicar:

1. Lê o texto atual do campo **Complemento**.
2. Normaliza e limpa o texto.
3. Acrescenta `INFODIP número/ano-UF` ao final, se ainda não constar.
4. Grava o resultado no campo e destaca o campo em verde.
5. Copia o resultado para a área de transferência.
6. Mostra um aviso com o que foi feito.

---

## Funções

### Normalização
- Converte o complemento para **maiúsculas**.
- Remove **acentos** e espaços repetidos.

### Limpeza de repetições
- **Município/UF repetido em sequência** passa a aparecer uma vez só.
  `FORTALEZA/CE/FORTALEZA/CE` → `FORTALEZA/CE`
- **Número da comunicação repetido** (incluindo a forma antiga `n/a/n/a`): mantém apenas a primeira ocorrência.

### Remoção de INFODIP anterior
Remove o que já veio no complemento antes de inserir o INFODIP atual:
- `INFODIP número/ano-UF`, inclusive de **outra comunicação**.
- `INFODIP número` **sem ano** (por exemplo, `INFODIP 4`).
- A palavra `INFODIP` **solta**, sem número depois.
- Separadores que sobraram (vírgulas, ponto e vírgula, barras e traços nas pontas).

A palavra só é removida quando aparece isolada. Textos como `MINFODIPX` não são alterados.

### Inclusão do INFODIP atual
- Acrescenta ` INFODIP número/ano-UF` no final do complemento.
- Se o número da comunicação já aparece no texto, não duplica.
- Se houver mais de um número diferente no mesmo modal, o botão aparece ao lado do campo **Complemento**. Ao clicar, o script exibe um aviso e preserva o campo e a área de transferência. Repetições do mesmo número contam como uma única comunicação.

### Limite de caracteres
- Respeita o limite de **70 caracteres**, ou o `maxlength` do campo, se for menor.
- Se o texto passar do limite, remove **pontos e traços** do texto anterior para tentar caber, e **avisa** que isso aconteceu.
- Se mesmo assim passar, **não altera o campo** e informa quantos caracteres precisam ser removidos.

### Cópia para a área de transferência
- Usa `GM.setClipboard` e **aguarda a conclusão** antes de mostrar o aviso de sucesso.
- Se o recurso do Tampermonkey não estiver disponível, usa `navigator.clipboard` como reserva.
- Se o campo for editado durante a cópia, **preserva a sua edição** e avisa.

### Proteções
- Bloqueia cliques repetidos enquanto a operação está em andamento.
- Confere novamente o campo e os números visíveis dentro do modal **no momento do clique**, inclusive quando a tela é atualizada.
- Na interface antiga, reconhece o bloco do campo pelo botão **Processar Comunicação no Sistema ELO** ou pelo formulário do próprio campo.
- Não age nas páginas de consulta (lista abaixo).

---

## Exemplos

Considere a comunicação `269/2026-CE`.

| Complemento antes | Complemento depois |
|---|---|
| *(vazio)* | `INFODIP 269/2026-CE` |
| `TERMO 1/CRC` | `TERMO 1/CRC INFODIP 269/2026-CE` |
| `termo 1/crc` | `TERMO 1/CRC INFODIP 269/2026-CE` |
| `TERMO 7055/CRC/FORTALEZA/CE/FORTALEZA/CE/ INFODIP 4` | `TERMO 7055/CRC/FORTALEZA/CE INFODIP 269/2026-CE` |
| `TERMO 7055/CRC/FORTALEZA/CE/FORTALEZA/CE INFODIP` | `TERMO 7055/CRC/FORTALEZA/CE INFODIP 269/2026-CE` |
| `TERMO 1/CRC INFODIP 12/2024-CE` | `TERMO 1/CRC INFODIP 269/2026-CE` |
| `TERMO 1/CRC INFODIP 269/2026-CE` | `TERMO 1/CRC INFODIP 269/2026-CE` *(já constava; apenas normalizado e copiado)* |

Exemplo de compactação, com campo limitado a 33 caracteres:

| Antes | Depois |
|---|---|
| `TERMO 1.2/CRC-X` | `TERMO 12/CRCX INFODIP 269/2026-CE` *(pontos e traços removidos do texto anterior)* |

---

## Avisos exibidos

| Situação | Aviso |
|---|---|
| Complemento preenchido | ✔️ Complemento preenchido e copiado! |
| Comunicação já constava | ℹ️ A comunicação já consta no complemento. Texto normalizado e copiado. |
| Repetição removida | ✔️ Repetição removida. Complemento atualizado e copiado! |
| INFODIP anterior removido | ✔️ INFODIP anterior removido. Complemento atualizado e copiado! |
| Pontuação removida para caber | ⚠️ Complemento copiado, mas pontos e traços foram removidos do texto anterior para caber em N caracteres. Confira o resultado. |
| Acima do limite | ⚠️ Excede o limite de N caracteres, mesmo sem pontos e traços. Remova X caracteres. |
| Falha na cópia | ⚠️ Complemento preenchido, mas não foi possível copiar para a área de transferência. |
| Campo editado durante a cópia | ⚠️ Sua edição foi preservada; a área de transferência pode conter o texto anterior. |
| Campo indisponível | ⚠️ O campo complemento não está disponível para edição. |
| Número não identificado | ⚠️ Não foi possível identificar com segurança o número da comunicação. |
| Mais de uma comunicação no modal | ⚠️ Mais de uma comunicação foi encontrada neste modal. Abra o modal da comunicação desejada e tente novamente. |
| Erro inesperado | ⚠️ Não foi possível concluir a operação. Confira o complemento antes de tentar novamente. |

Os avisos de sucesso somem em cerca de 2,5 segundos. Os alertas e erros ficam por cerca de 4,5 segundos.

---

## Instalação

1. Instale o **Tampermonkey** no navegador.
2. Na [página do script no Greasy Fork](https://greasyfork.org/pt-BR/scripts/542953), clique em **Instalar este script**.
3. Abra o INFODIP e, no modal da comunicação, use o botão **Copiar e Colar**.

## Arquivos do repositório

- [Script .js](infodip-copiar-e-colar-complemento.user.js) · [URL do arquivo bruto para sincronizar com o Greasy Fork](https://raw.githubusercontent.com/wiltonramon/infodip-copiar-e-colar-complemento/main/infodip-copiar-e-colar-complemento.user.js)
- [Cópia em .txt](infodip-copiar-e-colar-complemento.user.txt) com o mesmo conteúdo, para copiar e colar no editor do gerenciador de scripts.

---

## Onde funciona

O script roda em:

- `https://infodip.tse.jus.br/infodip/*`
- `https://infodiphmg.tse.jus.br/infodip/*`

Ele **não** é executado nestas páginas de consulta:

- `/infodip/comunicacao/consulta/`
- `/infodip/comunicacao/consulta/index.jsp`
- `/infodip/comunicacao/consulta/detalhes.jsp`
- `/infodip/comunicacao/consulta/comunicacao.jsp`

---

## Detalhes técnicos

O código é organizado em funções pequenas, com nomes em português.

### Lógica de texto (funções puras)

| Função | O que faz |
|---|---|
| `escaparRegExp` | Escapa caracteres especiais para uso em expressões regulares. |
| `normalizar` | Remove acentos, ajusta espaços e converte para maiúsculas. |
| `chaveComunicacao` | Gera a chave de comparação de um `número/ano[-UF]`. |
| `listarComunicacoes` | Localiza todos os números de comunicação em um texto, com posição. |
| `contemComunicacao` | Verifica se o número atual já aparece no texto. |
| `colapsarLocalidade` | Reduz `MUNICÍPIO/UF` repetido em sequência. |
| `manterPrimeira` | Mantém só a primeira ocorrência do número atual. |
| `removerInfodip` | Remove INFODIP anterior (com ou sem número) e a palavra solta. |
| `preparar` | Executa a lista `ETAPAS` e guarda o resultado de cada etapa. |
| `montar` | Acrescenta o INFODIP atual e aplica o limite de caracteres. |
| `classificar` | Define o tipo de resultado para escolher o aviso. |
| `construirComplemento` | Junta tudo e devolve o texto final, o tipo e se houve compactação. |

As etapas da preparação ficam na lista `ETAPAS`, em ordem: `normalizar`, `colapsarLocalidade`, `manterPrimeira` e `removerInfodip`. Para incluir ou reordenar uma regra, basta alterar essa lista.

### Página e interface

| Função | O que faz |
|---|---|
| `limiteDeCaracteres` | Calcula o limite efetivo (70 ou o `maxlength` do campo). |
| `localizarContexto` | Identifica o modal, o bloco da comunicação na interface antiga ou o formulário do campo. |
| `localizarCampo` | Encontra o campo Complemento visível e único no contexto identificado. |
| `localizarComunicacoes` | Identifica os números distintos e visíveis no mesmo contexto. |
| `exibirAviso` / `ocultarAviso` | Mostram e escondem o aviso na camada superior do navegador (`popover`). |
| `destacarCampo` | Destaca o campo em verde sem deixar a cor presa. |
| `definirValor` | Grava o valor e dispara os eventos `input` e `change`. |
| `copiarTexto` | Copia via `GM.setClipboard`, com `navigator.clipboard` de reserva. |
| `aoClicar` | Fluxo completo do clique no botão. |
| `sincronizarBotoes` | Posiciona um botão por modal, junto ao número identificado ou junto ao campo em caso de ambiguidade. |
| `agendarSincronizacao` | Agrupa as mudanças da página em uma verificação por quadro de animação. |

---

## Observações

- O script depende da estrutura do modal ou formulário, de `span.fontBlackBold` (número da comunicação) e de `#complemento` (campo). Se o TSE alterar o HTML da tela, o botão pode deixar de aparecer.
- Confira sempre o complemento antes de enviar, principalmente quando o aviso indicar remoção de pontuação.
- Este é um projeto independente e **não é oficial do TSE**.
