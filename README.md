# Telenova Webphone

Documentação oficial para configurar e integrar o Telenova Webphone em CRMs, ERPs e outras aplicações web.

## Por onde começar

Siga esta ordem para concluir uma integração:

1. Obtenha a API Key do tenant em [Autenticação](Docs/Autenticação.md).
2. Associe o usuário do seu sistema a um ramal em [Referência da API](Docs/API.md).
3. Carregue o Webphone por JavaScript ou iframe em [Integração](Docs/Integração.md).
4. Se algo não funcionar, consulte [Solução de problemas](Docs/Troubleshooting.md).

## Conceitos essenciais

- `userId` é o identificador do usuário no sistema integrado. Ele não é o número do ramal.
- `extension` é o ramal de telefonia associado ao `userId`.
- A API Key identifica o tenant. Não envie `domainUuid` nos endpoints públicos.
- O domínio da Fusion API e o domínio do Webphone podem ser diferentes.

Nos exemplos, substitua:

| Placeholder | Valor esperado |
| --- | --- |
| `<FUSION_API_URL>` | URL base da Fusion API, sem barra no final |
| `<WEBPHONE_URL>` | URL base do servidor Webphone, sem barra no final |
| `<API_KEY>` | API Key do tenant, iniciada por `wp_tk_` |
| `<USER_ID>` | Identificador do usuário no sistema integrado |
| `<JWT>` | Token administrativo obtido no login da Fusion API |

## Endpoints oficiais

A especificação importável está disponível em [OpenAPI](Docs/openapi.yaml).

### Fusion API

| Método | Endpoint | Autenticação | Finalidade |
| --- | --- | --- | --- |
| `GET` | `/fusionapi/webphone/configs` | API Key | Listar configurações |
| `GET` | `/fusionapi/webphone/configs/:userId` | API Key | Consultar uma configuração |
| `PUT` | `/fusionapi/webphone/configs/:userId` | API Key | Criar ou substituir uma configuração |
| `PATCH` | `/fusionapi/webphone/configs/:userId` | API Key | Alterar parte de uma configuração |
| `POST` | `/fusionapi/webphone/configs/:userId/change-id` | API Key | Trocar o `userId` |
| `POST` | `/fusionapi/auth` | Credenciais do painel | Obter JWT administrativo |
| `GET` | `/fusionapi/webphone/integration-key` | JWT | Consultar a API Key do tenant |

### Webphone

| Método | Endpoint | Finalidade |
| --- | --- | --- |
| `GET` | `/api/webphone` | Integração por JavaScript |
| `GET` | `/embed` | Integração por iframe |
| `GET` | `/demo` | Demonstração pública |

Somente os endpoints listados acima fazem parte desta documentação pública.

## Escopo

Esta documentação cobre autenticação, configuração, integração, API JavaScript, parâmetros de interface, erros e diagnóstico.

## Suporte

Antes de solicitar suporte, execute o roteiro de [Solução de problemas](Docs/Troubleshooting.md). Se o problema persistir, envie as informações indicadas na seção “O que enviar ao suporte”; isso evita novas rodadas de perguntas.
