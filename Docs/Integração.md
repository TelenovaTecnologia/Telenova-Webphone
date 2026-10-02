# Integração

O Telenova Webphone pode ser integrado de duas formas:

1. **JavaScript** — recomendado para aplicações que podem adicionar uma tag `<script>`;
2. **iframe** — indicado quando o Webphone precisa ficar isolado em uma página própria.

Nos dois modos, `userId` identifica o usuário do sistema integrado e a API Key precisa pertencer ao mesmo tenant da configuração desse usuário.

> [!IMPORTANT]
> No modelo atual, a API Key é enviada ao navegador e também autoriza a API de configuração do tenant. Não grave a chave no repositório, não a envie para analytics e não a registre em logs. Restrinja quem pode visualizar o código e as requisições da integração sempre que isso for possível.

## Pré-requisitos

Antes de integrar, confirme que:

- a aplicação e o Webphone são servidos por HTTPS;
- você possui a URL do Webphone e a API Key do tenant;
- existe uma configuração ativa para o `userId` informado;
- o navegador tem permissão para usar microfone e, quando necessário, câmera;
- a rede do usuário permite as conexões WSS e WebRTC utilizadas pela telefonia.

## Opção 1: JavaScript

Esta é a forma mais simples de carregar o Webphone. Troque os três valores destacados e adicione a tag antes do fechamento de `</body>`:

```html
<script
  src="https://<DOMINIO_DO_WEBPHONE>/api/webphone?userId=<USER_ID>&token=<API_KEY>"
></script>
```

Exemplo completo:

```html
<!doctype html>
<html lang="pt-BR">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Minha aplicação</title>
  </head>
  <body>
    <main>Conteúdo da aplicação</main>

    <script
      src="https://webphone.exemplo.com/api/webphone?userId=crm-user-123&token=wp_tk_EXEMPLO&startAsButton=true"
    ></script>
  </body>
</html>
```

Esse formato evita o uso de `eval`. Como a chave fica na URL, ela pode aparecer no histórico de rede, em logs de proxy e em ferramentas de observabilidade. Consulte [Cuidados com a API Key](Autenticação.md#cuidados-com-a-api-key).

### Carregamento usando o header Authorization

O endpoint também aceita:

```http
GET /api/webphone?userId=<USER_ID>
Authorization: Bearer <API_KEY>
```

Esse modo exige que a aplicação baixe e execute o JavaScript retornado. Use-o apenas quando a política de segurança da aplicação já possuir um carregador de scripts aprovado. Para a integração mais direta, use a tag `<script>` do exemplo anterior.

## Opção 2: iframe

```html
<iframe
  id="webphone-frame"
  src="https://<DOMINIO_DO_WEBPHONE>/embed?userId=<USER_ID>&token=<API_KEY>"
  allow="microphone; camera; autoplay; display-capture"
  title="Webphone"
  style="width: 330px; height: 560px; border: 0;"
></iframe>
```

O atributo `allow` libera para o iframe os recursos de navegador usados pelo Webphone. O usuário ainda pode precisar conceder permissão quando o navegador solicitar.

> [!WARNING]
> A API Key é enviada na query string do iframe. Evite incluir a URL completa em logs, prints, tickets ou ferramentas de analytics.

## API JavaScript

Na integração por JavaScript, a API pública fica disponível em `window.WP`.

### Aguardar o carregamento

A tag externa termina de carregar antes de todos os módulos internos do Webphone. Se a aplicação precisar discar automaticamente, aguarde a função pública ficar disponível:

```js
async function waitForWebphone(timeoutMs = 15000) {
  const startedAt = Date.now();

  while (typeof window.WP?.call !== 'function') {
    if (Date.now() - startedAt >= timeoutMs) {
      throw new Error('O Webphone não ficou pronto dentro do tempo esperado.');
    }
    await new Promise((resolve) => setTimeout(resolve, 100));
  }

  return window.WP;
}
```

Esse teste confirma que a API JavaScript foi carregada. O registro SIP ainda depende das credenciais, da rede e da infraestrutura de telefonia.

### Iniciar uma chamada

```js
const WP = await waitForWebphone();
const sessionId = WP.call('6533658182');

if (!sessionId) {
  console.error('Não foi possível iniciar a chamada.');
}
```

`WP.call(number)` recebe uma string e retorna o identificador da sessão ou `null` quando a chamada não pode ser iniciada.

### Encerrar uma chamada

```js
window.WP.hangup(sessionId);
```

Quando `sessionId` não é informado, o Webphone tenta encerrar todas as sessões conectadas encontradas pelo agente:

```js
window.WP.hangup();
```

### Remover o Webphone

```js
window.WP.destroy();
```

`destroy()` encerra recursos do agente e remove o widget. Use-o quando a aplicação desmontar a tela que contém o Webphone.

## Discar a partir do iframe

No iframe, `window.WP` pertence ao documento interno. A aplicação hospedeira deve enviar uma mensagem para a origem exata do Webphone:

```js
const frame = document.getElementById('webphone-frame');
const webphoneOrigin = 'https://webphone.exemplo.com';

frame.contentWindow.postMessage(
  {
    action: 'webphone:dial',
    number: '6533658182'
  },
  webphoneOrigin
);
```

Não use `"*"` como `targetOrigin`. A origem precisa ser exatamente a parte `protocolo + domínio + porta` da URL do Webphone.

O Webphone valida a origem usando a página que carregou o iframe. Uma política de `Referrer-Policy` que remova completamente o referrer pode impedir o comando; nesse caso, preserve ao menos a origem.

## Parâmetros de interface

As opções persistidas são definidas em `uiConfig`. Os mesmos nomes podem ser enviados na URL para sobrescrever aquela inicialização.

| Parâmetro | Tipo | Padrão quando ausente | Descrição |
| --- | --- | --- | --- |
| `startAsButton` | `boolean` | `false` | Inicia como botão flutuante |
| `showDialpad` | `boolean` | `false` | Exibe o teclado numérico |
| `showHistory` | `boolean` | `false` | Exibe o histórico |
| `showConfig` | `boolean` | `false` | Exibe as configurações locais |
| `showMute` | `boolean` | `false` | Exibe mute/unmute |
| `showHold` | `boolean` | `false` | Exibe hold/resume |
| `showTransfer` | `boolean` | `false` | Exibe transferência |
| `showDtmf` | `boolean` | `false` | Exibe o teclado DTMF durante a chamada |
| `tabGuard` | `boolean` | `false` | Mantém somente uma aba SIP ativa por ramal |
| `lockedNumber` | `string` | vazio | Preenche e bloqueia o número no dialpad |

Na URL, valores booleanos devem ser escritos como `true` ou `false`:

```html
<script
  src="https://webphone.exemplo.com/api/webphone?userId=crm-user-123&token=wp_tk_EXEMPLO&startAsButton=true&showDialpad=true"
></script>
```

Também existem três opções somente de inicialização:

| Parâmetro | Tipo | Descrição |
| --- | --- | --- |
| `themeColor` | `string` | Define a cor de destaque do tema |
| `customCssUrl` | `string` | Carrega uma folha de estilos adicional por HTTPS |
| `customCss` | `string` | Injeta CSS adicional; precisa ser codificado para uso na URL |

### Ordem de precedência

Quando a mesma opção aparece em mais de um lugar, vale esta ordem:

1. parâmetro enviado na URL;
2. valor persistido em `uiConfig`;
3. valor padrão do Webphone.

## Respostas do endpoint de runtime

| Situação | Status | Resposta |
| --- | ---: | --- |
| Carregamento válido | `200` | JavaScript do Webphone |
| API Key ausente | `403` | JavaScript que registra erro no console |
| `userId` ausente | `400` | JavaScript que registra erro no console |
| API Key inválida ou tenant incorreto | `403` | JavaScript de acesso negado |
| Webphone inativo | `403` | JavaScript de aviso |

No modo iframe, os erros são retornados como HTML. Um `userId` sem Webphone disponível retorna `404`.

## Demonstração

`GET https://<DOMINIO_DO_WEBPHONE>/demo` abre a demonstração pública. Ela serve para conhecer a interface e não substitui a homologação com o `userId`, a API Key e os ramais reais do tenant.

## Próximos passos

- [Autenticação](Autenticação.md)
- [Referência da API](API.md)
- [Solução de problemas](Troubleshooting.md)
