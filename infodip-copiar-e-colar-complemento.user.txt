// ==UserScript==
// @name         Botão Copiar e Colar complemento no INFODIP
// @namespace    https://greasyfork.org/pt-BR/scripts/542953
// @version      4.6.2
// @description  Preenche e copia o complemento em maiúsculas, reconhecendo a comunicação no modal (número/ano-UF), com limpeza de repetições, remoção de INFODIP anterior e cópia confirmada via GM.setClipboard.
// @author       Ramon Machado
// @match        https://infodip.tse.jus.br/infodip/*
// @match        https://infodiphmg.tse.jus.br/infodip/*
// @run-at       document-idle
// @grant        GM.setClipboard
// @icon         data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA2NCA2NCI+PHJlY3QgeD0iMTIiIHk9IjgiIHdpZHRoPSI0MCIgaGVpZ2h0PSI1MiIgcng9IjYiIGZpbGw9IiNjOGEyNmIiLz48cmVjdCB4PSIxOCIgeT0iMTYiIHdpZHRoPSIyOCIgaGVpZ2h0PSIzOCIgcng9IjIiIGZpbGw9IiNmZmYiLz48cmVjdCB4PSIyMiIgeT0iNCIgd2lkdGg9IjIwIiBoZWlnaHQ9IjEyIiByeD0iNCIgZmlsbD0iIzVmNmI3NiIvPjxwYXRoIGQ9Ik0yMyAyOGgxOE0yMyAzNWgxOE0yMyA0MmgxMiIgc3Ryb2tlPSIjOGE5NmEzIiBzdHJva2Utd2lkdGg9IjMiIHN0cm9rZS1saW5lY2FwPSJyb3VuZCIgZmlsbD0ibm9uZSIvPjwvc3ZnPgo=
// @license      GPL-3.0-or-later
// ==/UserScript==

(function () {
    'use strict';

    const LIMITE_MAXIMO = 70;
    const PAGINAS_EXCLUIDAS = [
        '/infodip/comunicacao/consulta/',
        '/infodip/comunicacao/consulta/index.jsp',
        '/infodip/comunicacao/consulta/detalhes.jsp',
        '/infodip/comunicacao/consulta/comunicacao.jsp'
    ];

    // Páginas excluídas: nada é criado nem observado.
    if (PAGINAS_EXCLUIDAS.includes(location.pathname.toLowerCase())) return;

    const UF = 'AC|AL|AP|AM|BA|CE|DF|ES|GO|MA|MT|MS|MG|PA|PB|PR|PE|PI|RJ|RN|RS|RO|RR|SC|SP|SE|TO';
    const REGEX_NUMERO_SPAN = new RegExp(`^\\d+/\\d{4}-(?:${UF})$`);

    const MENSAGENS = {
        sucesso: '✔️ Complemento preenchido e copiado!',
        duplicado: 'ℹ️ A comunicação já consta no complemento. Texto normalizado e copiado.',
        reparado: '✔️ Repetição removida. Complemento atualizado e copiado!',
        substituido: '✔️ INFODIP anterior removido. Complemento atualizado e copiado!',
        longoDemais: (quantidade, limite) => `⚠️ Excede o limite de ${limite} caracteres, mesmo sem pontos e traços. Remova ${quantidade} caracteres.`,
        pontuacaoRemovida: (limite) => `⚠️ Complemento copiado, mas pontos e traços foram removidos do texto anterior para caber em ${limite} caracteres. Confira o resultado.`,
        erroCopia: '⚠️ Complemento preenchido, mas não foi possível copiar para a área de transferência.',
        editado: '⚠️ Sua edição foi preservada; a área de transferência pode conter o texto anterior.',
        semNumero: '⚠️ Não foi possível identificar com segurança o número da comunicação.',
        variasComunicacoes: '⚠️ Mais de uma comunicação foi encontrada neste modal. Abra o modal da comunicação desejada e tente novamente.',
        semCampo: '⚠️ O campo complemento não está disponível para edição.',
        inesperado: '⚠️ Não foi possível concluir a operação. Confira o complemento antes de tentar novamente.'
    };

    /* ---------- Lógica de texto (funções puras) ---------- */

    const escaparRegExp = (texto) => texto.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');

    const normalizar = (texto) => String(texto ?? '').normalize('NFD')
        .replace(/[\u0300-\u036f]/g, '')
        .replace(/\s+/g, ' ').trim().toUpperCase();

    const chaveComunicacao = (texto) => {
        const achado = normalizar(texto).match(new RegExp(`^(\\d+(?:\\.\\d+)*)\\s*/\\s*(\\d{4})(?:\\s*-?\\s*(${UF}))?$`));
        return achado ? `${achado[1].replace(/\./g, '')}/${achado[2]}${achado[3] ? '-' + achado[3] : ''}` : null;
    };

    // Todos os "número/ano[-UF]" do texto, com posição.
    const listarComunicacoes = (texto) => {
        const expressao = new RegExp(
            `(^|[^A-Z0-9_./-])(\\d+(?:\\.\\d+)*\\s*/\\s*\\d{4}(?:\\s*-?\\s*(?:${UF}))?)(?![A-Z0-9_/-]|\\.\\d)`, 'g');
        const lista = [];
        let achado;
        while ((achado = expressao.exec(texto)) !== null) {
            const chave = chaveComunicacao(achado[2]);
            if (chave) lista.push({ chave, inicio: achado.index + achado[1].length, fim: achado.index + achado[0].length });
        }
        return lista;
    };

    const contemComunicacao = (texto, numero) => {
        const chave = chaveComunicacao(numero);
        return Boolean(chave && listarComunicacoes(normalizar(texto)).some(item => item.chave === chave));
    };

    // MUNICÍPIO/UF repetido em sequência.
    const colapsarLocalidade = (texto) => texto.replace(
        new RegExp(`(^|[^A-Z0-9_])([A-Z](?:[A-Z .'-]*[A-Z])?)\\s*/\\s*(${UF})(?:\\s*/\\s*\\2\\s*/\\s*\\3)+(?![A-Z0-9_])`, 'g'),
        '$1$2/$3');

    // Mantém só a primeira ocorrência do número atual.
    const manterPrimeira = (texto, numero) => {
        const chave = chaveComunicacao(numero);
        if (!chave) return texto;
        const exato = escaparRegExp(numero);
        let resultado = texto.replace(
            new RegExp(`(^|[^A-Z0-9_./-])(${exato})(?:\\s*/\\s*${exato})+(?![A-Z0-9_/-]|\\.\\d)`, 'g'), '$1$2');
        const iguais = listarComunicacoes(resultado).filter(ocorrencia => ocorrencia.chave === chave);
        for (let i = iguais.length - 1; i >= 1; i--) {
            const ocorrencia = iguais[i];
            const rotulo = resultado.slice(0, ocorrencia.inicio).match(/\bINFODIP\s*$/);
            const inicio = rotulo ? ocorrencia.inicio - rotulo[0].length : ocorrencia.inicio;
            let esquerda = resultado.slice(0, inicio);
            let direita = resultado.slice(ocorrencia.fim);
            if (/^[\s,;]*$/.test(direita)) { esquerda = esquerda.replace(/[\s,;]+$/, ''); direita = ''; }
            else if (/[\s,;]$/.test(esquerda) && /^[\s,;]/.test(direita)) direita = direita.replace(/^[\s,;]+/, ' ');
            resultado = esquerda + direita;
        }
        return resultado.replace(/\s+/g, ' ').trim();
    };

    // Remove qualquer "INFODIP número[/ano[-UF]]" (o ano é opcional: "INFODIP 4" também sai)
    // e também a palavra INFODIP solta, sem número ("... FORTALEZA/CE INFODIP").
    // UF sem hífen só é removida no fim do texto.
    const removerInfodip = (texto) => {
        const expressao = new RegExp(
            `(^|[^A-Z0-9_])INFODIP(?![A-Z_])\\s*:?\\s*\\d+(?:\\.\\d+)*` +
            `(?:\\s*/\\s*\\d{4}(?:\\s*-\\s*(?:${UF})(?![A-Z0-9_])|\\s+(?:${UF})(?=\\s*$))?)?` +
            `(?![A-Z0-9_/-]|\\.\\d)`, 'g');
        const semInfodip = texto
            .replace(expressao, '$1')
            .replace(/(^|[^A-Z0-9_])INFODIP(?![A-Z0-9_])\s*:?/g, '$1');
        if (semInfodip === texto) return texto;
        return semInfodip
            .replace(/\s*([,;])(?:\s*[,;])+/g, '$1')
            .replace(/\s+/g, ' ')
            .replace(/^[\s,;:\-/]+|[\s,;:\-/]+$/g, '');
    };

    // Etapas de preparação do texto, em ordem. Cada etapa recebe o texto da anterior
    // e o número da comunicação. Para incluir ou reordenar uma regra, mexa só nesta lista.
    const ETAPAS = [
        { nome: 'normalizar',          executar: (texto) => normalizar(texto) },
        { nome: 'colapsarLocalidade', executar: (texto) => colapsarLocalidade(texto) },
        { nome: 'manterPrimeira',     executar: (texto, numero) => manterPrimeira(texto, numero) },
        { nome: 'removerInfodip',     executar: (texto) => removerInfodip(texto) }
    ];

    // Executa as etapas e guarda o texto resultante de cada uma, pelo nome.
    function preparar(textoBruto, numero) {
        const resultadosEtapas = {};
        let texto = textoBruto;
        for (const etapa of ETAPAS) {
            texto = etapa.executar(texto, numero);
            resultadosEtapas[etapa.nome] = texto;
        }
        return resultadosEtapas;
    }

    // Acrescenta o INFODIP atual (se ainda não consta) e aplica o limite de caracteres.
    // Se precisar, remove pontos e traços do texto anterior. Devolve { texto, compactado } ou { excedente }.
    function montar(base, numero, limite) {
        const presente = contemComunicacao(base, numero);
        const comporTexto = (parte) => presente ? parte : `${parte} INFODIP ${numero}`.trim();

        let texto = comporTexto(base);
        let compactado = false;
        if (texto.length > limite) {
            const compacto = base.replace(/[.\-]/g, '');
            compactado = compacto !== base;
            texto = comporTexto(compacto);
        }
        if (texto.length > limite) return { excedente: texto.length - limite };
        return { texto, compactado, presente };
    }

    // Classifica o que aconteceu, para escolher a mensagem.
    function classificar(resultadosEtapas, texto, presente) {
        const original = resultadosEtapas.normalizar;
        const limpo = resultadosEtapas.manterPrimeira;
        const removido = resultadosEtapas.removerInfodip !== limpo;
        if (removido) return texto === original ? 'duplicado' : 'substituido';
        if (limpo !== original) return 'reparado';
        return presente ? 'duplicado' : 'sucesso';
    }

    // Devolve { texto, tipo, compactado } ou { excedente }.
    function construirComplemento(textoBruto, numero, limite) {
        const resultadosEtapas = preparar(textoBruto, numero);
        const resultado = montar(resultadosEtapas.removerInfodip, numero, limite);
        if (resultado.excedente) return { excedente: resultado.excedente };
        return {
            texto: resultado.texto,
            tipo: classificar(resultadosEtapas, resultado.texto, resultado.presente),
            compactado: resultado.compactado
        };
    }

    /* ---------- DOM ---------- */

    const SELETOR_MODAL = 'dialog, [role="dialog"], [role="alertdialog"], [aria-modal="true"], .modal, .ui-dialog, .rich-modalpanel, .rf-pp';
    const SELETOR_CAMPO = 'textarea[id="complemento"], input[id="complemento"]';
    const SELETOR_ACAO_ELO = '[id="Processar Comunicação no Sistema ELO"]';
    const ID_BOTAO = 'btnInfodipRemoverDuplicacao';

    const limiteDeCaracteres = (campo) =>
        Number.isInteger(campo.maxLength) && campo.maxLength >= 0
            ? Math.min(LIMITE_MAXIMO, campo.maxLength) : LIMITE_MAXIMO;

    function estaVisivel(elemento) {
        return !elemento.closest('[hidden], [aria-hidden="true"], [inert]') &&
            elemento.getClientRects().length > 0 &&
            getComputedStyle(elemento).visibility === 'visible';
    }

    function localizarNoContexto(contexto, seletor) {
        const modal = contexto.closest(SELETOR_MODAL);
        return Array.from(contexto.querySelectorAll(seletor)).filter(elemento =>
            elemento.closest(SELETOR_MODAL) === modal && estaVisivel(elemento)
        );
    }

    function localizarContexto(campo) {
        const modal = campo.closest(SELETOR_MODAL);
        if (modal) return modal;

        // Na interface antiga, o controle do ELO identifica o bloco da comunicação.
        // Para no primeiro bloco que contém o campo e a ação, mesmo se o número estiver ausente.
        for (let bloco = campo.parentElement; bloco && bloco !== document.body; bloco = bloco.parentElement) {
            const acoes = localizarNoContexto(bloco, SELETOR_ACAO_ELO);
            if (acoes.length === 1) return bloco;
            if (acoes.length > 1) break;
        }

        return campo.closest('form');
    }

    function localizarCampo(contexto) {
        const campos = localizarNoContexto(contexto, SELETOR_CAMPO);
        return campos.length === 1 ? campos[0] : null;
    }

    function localizarComunicacoes(contexto) {
        const comunicacoes = new Map();
        for (const elemento of localizarNoContexto(contexto, 'span.fontBlackBold')) {
            const numero = elemento.textContent.trim().toUpperCase();
            if (REGEX_NUMERO_SPAN.test(numero) && !comunicacoes.has(numero)) {
                comunicacoes.set(numero, { elemento, numero });
            }
        }
        return Array.from(comunicacoes.values());
    }

    // Aviso na camada superior (popover): sem z-index nem recorte por modal.
    const aviso = document.createElement('div');
    aviso.setAttribute('role', 'status');
    aviso.setAttribute('aria-live', 'polite');
    aviso.setAttribute('popover', 'manual');
    Object.assign(aviso.style, {
        position: 'fixed', inset: 'auto', margin: '0', padding: '8px 10px',
        width: '360px', maxWidth: 'calc(100vw - 16px)', maxHeight: '140px', overflowY: 'auto',
        boxSizing: 'border-box', background: '#fff', border: '1px solid #aab4bd',
        borderRadius: '5px', boxShadow: '0 2px 8px rgba(0,0,0,.18)',
        font: 'bold 12px/1.4 Arial, sans-serif', overflowWrap: 'anywhere', zIndex: '2147483647'
    });
    document.body.appendChild(aviso);

    const temPopover = typeof aviso.showPopover === 'function';
    let temporizadorAviso = 0;

    function ocultarAviso() {
        if (temPopover) { if (aviso.matches(':popover-open')) aviso.hidePopover(); }
        else aviso.style.display = 'none';
    }

    function exibirAviso(mensagem, tipo, ancora) {
        aviso.textContent = mensagem;
        aviso.style.color = tipo === 'erro' ? '#d32f2f' : tipo === 'alerta' ? '#8a5700' : '#2e7d32';
        if (temPopover) { if (!aviso.matches(':popover-open')) aviso.showPopover(); }
        else aviso.style.display = 'block';

        const larguraJanela = document.documentElement.clientWidth, alturaJanela = document.documentElement.clientHeight;
        const retangulo = ancora.getBoundingClientRect();
        const largura = Math.min(360, larguraJanela - 16), altura = aviso.offsetHeight;
        let topo = retangulo.bottom + 8;
        if (topo + altura > alturaJanela - 8) topo = retangulo.top - 8 - altura;
        aviso.style.left = `${Math.max(8, Math.min(retangulo.left, larguraJanela - largura - 8))}px`;
        aviso.style.top = `${Math.max(8, topo)}px`;

        clearTimeout(temporizadorAviso);
        temporizadorAviso = setTimeout(ocultarAviso, tipo === 'sucesso' ? 2500 : 4500);
    }

    const estadoDestaque = new WeakMap();

    function destacarCampo(campo) {
        const anterior = estadoDestaque.get(campo);
        const original = anterior ? anterior.original : campo.style.backgroundColor;
        if (anterior) clearTimeout(anterior.temporizador);
        campo.style.backgroundColor = '#b2f2bb';
        const temporizador = setTimeout(() => {
            campo.style.backgroundColor = original;
            estadoDestaque.delete(campo);
        }, 2500);
        estadoDestaque.set(campo, { original, temporizador });
    }

    function definirValor(campo, texto) {
        if (campo.value === texto) return false;
        campo.value = texto;
        campo.dispatchEvent(new Event('input', { bubbles: true }));
        campo.dispatchEvent(new Event('change', { bubbles: true }));
        return true;
    }

    async function copiarTexto(texto) {
        try {
            if (typeof GM !== 'undefined' && typeof GM.setClipboard === 'function') {
                await GM.setClipboard(texto, 'text'); // só retorna após a conclusão
                return true;
            }
        } catch (erro) { /* tenta a API do navegador */ }
        try {
            await navigator.clipboard.writeText(texto);
            return true;
        } catch (erro) {
            return false;
        }
    }

    let ocupado = false;

    async function aoClicar(botao, contexto) {
        if (ocupado || !botao.isConnected || !contexto.contains(botao)) return;
        ocupado = true;
        botao.disabled = true;
        try {
            const campo = localizarCampo(contexto);
            if (!campo || campo.disabled || campo.readOnly || localizarContexto(campo) !== contexto) {
                return exibirAviso(MENSAGENS.semCampo, 'erro', botao);
            }
            const comunicacoes = localizarComunicacoes(contexto);
            if (comunicacoes.length > 1) return exibirAviso(MENSAGENS.variasComunicacoes, 'alerta', botao);
            const [comunicacao] = comunicacoes;
            if (!comunicacao) return exibirAviso(MENSAGENS.semNumero, 'erro', botao);

            const limite = limiteDeCaracteres(campo);
            const resultado = construirComplemento(campo.value, comunicacao.numero, limite);
            if (resultado.excedente) return exibirAviso(MENSAGENS.longoDemais(resultado.excedente, limite), 'erro', botao);

            if (definirValor(campo, resultado.texto)) destacarCampo(campo);
            const copiado = await copiarTexto(resultado.texto);

            if (campo.value !== resultado.texto) return exibirAviso(MENSAGENS.editado, 'alerta', botao);
            if (!copiado) return exibirAviso(MENSAGENS.erroCopia, 'alerta', botao);
            if (resultado.compactado) return exibirAviso(MENSAGENS.pontuacaoRemovida(limite), 'alerta', botao);
            exibirAviso(MENSAGENS[resultado.tipo], 'sucesso', botao);
        } catch (erro) {
            console.error('[INFODIP Script]', erro);
            exibirAviso(MENSAGENS.inesperado, 'erro', botao);
        } finally {
            setTimeout(() => { ocupado = false; botao.disabled = false; }, 600);
        }
    }

    function criarBotao(contexto) {
        const botao = document.createElement('button');
        botao.type = 'button';
        botao.textContent = 'Copiar e Colar';
        Object.assign(botao.style, {
            marginLeft: '10px', padding: '4px 8px', fontSize: '12px',
            cursor: 'pointer', fontWeight: 'bold'
        });
        botao.addEventListener('click', () => { void aoClicar(botao, contexto); });
        return botao;
    }

    const botoesDosContextos = new Map();

    function sincronizarBotoes() {
        const contextosComBotao = new Set();
        for (const campo of document.querySelectorAll(SELETOR_CAMPO)) {
            if (!estaVisivel(campo)) continue;
            const contexto = localizarContexto(campo);
            if (!contexto || localizarCampo(contexto) !== campo) continue;

            const comunicacoes = localizarComunicacoes(contexto);
            if (comunicacoes.length === 0) continue;

            const ancora = comunicacoes.length === 1 ? comunicacoes[0].elemento : campo;
            const botao = botoesDosContextos.get(contexto) ?? criarBotao(contexto);
            if (botao.previousSibling !== ancora) ancora.after(botao);
            botoesDosContextos.set(contexto, botao);
            contextosComBotao.add(contexto);
        }

        for (const [contexto, botao] of botoesDosContextos) {
            if (!contextosComBotao.has(contexto)) {
                botao.remove();
                botoesDosContextos.delete(contexto);
            } else if (contextosComBotao.size === 1) {
                // Outro script do INFODIP usa este ID para se posicionar ao lado do botão.
                if (botao.id !== ID_BOTAO) botao.id = ID_BOTAO;
            } else if (botao.id === ID_BOTAO) {
                botao.removeAttribute('id');
            }
        }
    }

    let quadro = 0;
    const agendarSincronizacao = () => {
        if (quadro) return;
        quadro = requestAnimationFrame(() => {
            quadro = 0;
            try { sincronizarBotoes(); } catch (erro) { console.error('[INFODIP Script]', erro); }
        });
    };

    new MutationObserver(agendarSincronizacao).observe(document.documentElement, {
        childList: true,
        subtree: true,
        characterData: true,
        attributes: true,
        attributeFilter: ['class', 'id', 'role', 'open', 'hidden', 'aria-modal', 'aria-hidden', 'inert', 'style']
    });
    agendarSincronizacao();
})();
