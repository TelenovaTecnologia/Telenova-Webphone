# Solução de problemas

Use este roteiro na ordem apresentada. Ele separa problemas de configuração, carregamento no navegador, registro SIP e mídia.

## Diagnóstico rápido

| Sintoma | Verifique primeiro |
| --- | --- |
| Fusion API retorna `401` | Tipo e valor do token no header `Authorization` |
| Fusion API retorna `404` | `userId`, ramal e tenant da API Key |
| Fusion API retorna `409` | Associação existente do `userId` ou do ramal |
| Fusion API retorna `422` | Tipos, campos obrigatórios e propriedades desconhecidas |
| Script do Webphone retorna `400` | Presença do parâmetro `userId` |
| Script ou iframe retorna `403` | API Key, tenant e campo `active` |
| Iframe retorna `404` | Existência da configuração para o `userId` |
| Widget não aparece | Console, Network, CSP e carregamento dos assets |
| Widget aparece, mas não registra | Configuração ativa, WSS, certificado e rede |
| Chamada conecta sem áudio | Permissão de microfone, dispositivo, firewall e ICE |
| `postMessage` não disca | `id` do iframe, `targetOrigin`, referrer e carregamento |

## 1. Confirmar a configuração na Fusion API

Consulte exatamente o `userId` usado no navegador:

```bash
curl --request GET \
  --url '<FUSION_API_URL>/fusionapi/webphone/configs/<USER_ID>' \
  --header 'Authorization: Bearer <API_KEY>'
```

Confirme na resposta:

- `userId` é o mesmo enviado ao Webphone;
- `extension` é o ramal esperado;
- `active` é `true`;
- a API Key pertence ao tenant dessa configuração.

Se a resposta for `404`, não tente corrigir pelo navegador. Primeiro crie ou ajuste a configuração usando a [Referência da API](API.md).

## 2. Conferir a requisição no navegador

Abra as ferramentas de desenvolvedor e acesse a aba **Network**.

Na integração por JavaScript, localize:

```text
GET <WEBPHONE_URL>/api/webphone?userId=...&token=...
```

No iframe, localize:

```text
GET <WEBPHONE_URL>/embed?userId=...&token=...
```

Verifique o status HTTP sem copiar ou compartilhar a URL completa, pois ela contém a API Key.

| Status | Interpretação |
| ---: | --- |
| `200` | O servidor aceitou a solicitação; continue verificando console e assets |
| `400` | `userId` ausente ou inválido |
| `403` | Chave ausente/inválida, tenant incorreto ou Webphone inativo |
| `404` | Configuração não encontrada, especialmente no iframe |
| `500` | Falha no servidor; reúna os dados da seção de suporte |

## 3. Widget não aparece

No console do navegador, procure erros relacionados a:

- Content Security Policy (`script-src`, `style-src`, `font-src` ou `connect-src`);
- bloqueio de Mixed Content;
- falha no carregamento de JavaScript, CSS ou fontes;
- acesso negado ou parâmetro `userId` ausente.

Confirme também que:

- a tag `<script>` está antes de `</body>`;
- a URL usa HTTPS;
- `userId` e `token` foram codificados corretamente;
- extensões de bloqueio de conteúdo não estão interceptando a requisição;
- a aplicação não removeu o elemento durante uma troca de tela.

Se a aplicação usa uma CSP restritiva, permita explicitamente o domínio do Webphone nos tipos de recurso necessários. Não habilite `unsafe-eval`; o exemplo recomendado não depende de `eval`.

## 4. Widget aparece, mas não registra

Esse cenário indica que o HTML e os assets carregaram, mas a conexão de telefonia não foi concluída.

Verifique:

1. `active` está como `true` na Fusion API;
2. o ramal ainda existe e está habilitado no tenant;
3. o certificado do endpoint WSS é válido;
4. a rede permite WebSocket seguro;
5. não existe proxy ou firewall encerrando a conexão;
6. `tabGuard` não está bloqueando o mesmo ramal aberto em outra aba.

O sucesso de `/demo` prova apenas o carregamento básico da interface. Ele não comprova que o usuário real, o ramal e a infraestrutura SIP estão corretos.

## 5. Microfone, câmera ou áudio

Para problemas de mídia:

- confirme que a página e o Webphone usam HTTPS;
- confira as permissões do site no navegador;
- confirme que o dispositivo correto está selecionado no sistema operacional;
- feche aplicações que possam estar usando o dispositivo exclusivamente;
- no iframe, mantenha `allow="microphone; camera; autoplay; display-capture"`;
- teste em outra rede para separar problema de aplicação de bloqueio de firewall/NAT;
- faça uma chamada entre dois ramais de teste e valide áudio nos dois sentidos.

Uma chamada sinalizada como conectada não garante, sozinha, que a mídia WebRTC foi estabelecida.

## 6. `window.WP` não está disponível

Os módulos internos carregam de forma assíncrona. Não chame `window.WP.call()` imediatamente após criar a tag `<script>`.

Use `waitForWebphone()` do guia de [Integração](Integração.md#aguardar-o-carregamento). Se houver timeout:

1. confira o status de `/api/webphone`;
2. procure assets bloqueados na aba Network;
3. verifique erros de CSP no console;
4. confirme que o elemento não foi removido durante o carregamento.

## 7. `postMessage` do iframe não funciona

Confirme que:

- o iframe possui `id="webphone-frame"`;
- `frame.contentWindow` existe;
- `targetOrigin` contém exatamente protocolo, domínio e porta do Webphone;
- o segundo argumento de `postMessage` não é `"*"`;
- a mensagem usa `{ action: "webphone:dial", number: "..." }`;
- a política de referrer da página não remove completamente a origem;
- a mensagem é enviada depois do carregamento do Webphone.

## 8. Configuração visual não é aplicada

Verifique a precedência:

1. parâmetros da URL;
2. `uiConfig` persistido;
3. valores padrão.

Na URL, envie booleanos como texto `true` ou `false`. Na API JSON, envie booleanos reais, sem aspas.

Ao alterar `uiConfig` via `PATCH`, lembre-se de que o objeto enviado substitui todo o `uiConfig` anterior. Uma opção omitida volta ao comportamento padrão.

Para `customCssUrl`:

- use HTTPS;
- confirme que a URL pode ser acessada pelo navegador do usuário;
- confira bloqueios de CSP e Mixed Content;
- verifique erros de fontes ou recursos referenciados dentro do CSS.

## Homologação mínima

Antes de liberar a integração:

- [ ] a configuração pode ser consultada pelo `userId`;
- [ ] `active` está como `true`;
- [ ] o Webphone carrega sem erros no console;
- [ ] o ramal registra;
- [ ] uma chamada de saída completa;
- [ ] uma chamada de entrada toca e atende;
- [ ] áudio funciona nos dois sentidos;
- [ ] mute, hold, DTMF e transferência foram testados quando habilitados;
- [ ] iframe e `postMessage` foram testados, se utilizados;
- [ ] a aplicação remove o Webphone com `destroy()` quando necessário;
- [ ] nenhuma API Key real aparece em código versionado, logs ou analytics.

## O que enviar ao suporte

Se o problema continuar, envie em uma única solicitação:

- ambiente e URLs base, sem query string e sem API Key;
- `userId` afetado e tenant;
- data, hora e fuso horário do teste;
- navegador e versão;
- modo de integração: JavaScript ou iframe;
- status HTTP observado;
- código e mensagem de erro retornados, removendo tokens;
- passos exatos para reproduzir;
- captura do console com segredos ocultados;
- informação se o problema acontece com um usuário, um tenant ou todos;
- resultado da homologação mínima acima.

Nunca envie API Key, JWT, senha do painel ou credenciais SIP em tickets, prints ou mensagens.

## Próximos passos

- [Autenticação](Autenticação.md)
- [Referência da API](API.md)
- [Integração](Integração.md)
