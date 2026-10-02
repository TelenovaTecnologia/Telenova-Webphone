# Referência da API

Esta página documenta os endpoints oficiais da Fusion API usados para configurar o Telenova Webphone.

Para importar os contratos em Swagger, Postman ou ferramentas de geração de cliente, use o arquivo [OpenAPI](openapi.yaml).

## Base URL e autenticação

Todas as rotas desta seção usam a base:

```text
<FUSION_API_URL>/fusionapi
```

Envie a API Key do tenant em todas as requisições:

```http
Authorization: Bearer <API_KEY>
```

O tenant é derivado da chave. Não envie `domainUuid` na URL ou no body.

## Modelo público

```json
{
  "userId": "crm-user-123",
  "extension": "1001",
  "active": true,
  "uiConfig": {
    "showDialpad": true,
    "showHistory": true,
    "showMute": true,
    "showHold": true,
    "showTransfer": true,
    "showDtmf": true,
    "showConfig": false,
    "startAsButton": true,
    "lockedNumber": "",
    "tabGuard": false
  },
  "updatedAt": "2026-10-02T12:00:00.000Z"
}
```

| Campo | Tipo | Regra |
| --- | --- | --- |
| `userId` | `string` | Identificador do usuário no sistema integrado; 1 a 128 caracteres; único no tenant |
| `extension` | `string` | Ramal existente no mesmo tenant; 1 a 32 caracteres |
| `active` | `boolean` | Quando `false`, impede o bootstrap do Webphone |
| `uiConfig` | `object` | Objeto fechado; campos desconhecidos retornam `422` |
| `updatedAt` | `string` | Data da última alteração em formato ISO 8601 |

`userId` é opaco: pode ser UUID ou outro identificador estável do sistema integrado. Não use informação que possa mudar com frequência, como nome, e-mail ou número do ramal.

## `uiConfig`

| Campo | Tipo | Padrão quando ausente | Descrição |
| --- | --- | --- | --- |
| `showDialpad` | `boolean` | `false` | Exibe o teclado numérico |
| `showHistory` | `boolean` | `false` | Exibe o histórico |
| `showMute` | `boolean` | `false` | Exibe mute/unmute |
| `showHold` | `boolean` | `false` | Exibe hold/resume |
| `showTransfer` | `boolean` | `false` | Exibe transferência |
| `showDtmf` | `boolean` | `false` | Exibe DTMF durante a chamada |
| `showConfig` | `boolean` | `false` | Exibe configurações locais |
| `startAsButton` | `boolean` | `false` | Inicia como botão flutuante |
| `lockedNumber` | `string` | vazio | Preenche e bloqueia o número no dialpad |
| `tabGuard` | `boolean` | `false` | Mantém somente uma aba SIP ativa por ramal |

`lockedNumber` aceita até 64 caracteres. São permitidos dígitos, espaços e os símbolos telefônicos `+`, `*`, `#`, `(`, `)`, `.`, `-`.

## Listar configurações

```http
GET /fusionapi/webphone/configs
```

Exemplo:

```bash
curl --request GET \
  --url '<FUSION_API_URL>/fusionapi/webphone/configs' \
  --header 'Authorization: Bearer <API_KEY>'
```

Resposta `200`:

```json
{
  "items": [
    {
      "userId": "crm-user-123",
      "extension": "1001",
      "active": true,
      "uiConfig": {},
      "updatedAt": "2026-10-02T12:00:00.000Z"
    }
  ]
}
```

A versão atual retorna todas as configurações do tenant e não possui paginação.

## Consultar uma configuração

```http
GET /fusionapi/webphone/configs/:userId
```

Codifique o `userId` como segmento de URL. Em JavaScript, use `encodeURIComponent(userId)`.

```bash
curl --request GET \
  --url '<FUSION_API_URL>/fusionapi/webphone/configs/crm-user-123' \
  --header 'Authorization: Bearer <API_KEY>'
```

Resposta `200`: o [modelo público](#modelo-público).

Retorna `404 WEBPHONE_NOT_FOUND` quando o recurso não existe ou pertence a outro tenant.

## Criar ou substituir uma configuração

```http
PUT /fusionapi/webphone/configs/:userId
```

O `PUT` exige o objeto completo. Os campos `extension`, `active` e `uiConfig` são obrigatórios, mesmo quando `uiConfig` for vazio.

```bash
curl --request PUT \
  --url '<FUSION_API_URL>/fusionapi/webphone/configs/crm-user-123' \
  --header 'Authorization: Bearer <API_KEY>' \
  --header 'Content-Type: application/json' \
  --data '{
    "extension": "1001",
    "active": true,
    "uiConfig": {
      "showDialpad": true,
      "showHistory": true,
      "showMute": true,
      "showHold": true,
      "showTransfer": true,
      "showDtmf": true,
      "showConfig": false,
      "startAsButton": true,
      "lockedNumber": "",
      "tabGuard": false
    }
  }'
```

| Status | Resultado |
| ---: | --- |
| `201` | Configuração criada |
| `200` | Configuração existente substituída |
| `404` | Ramal não encontrado no tenant |
| `409` | `userId` ou ramal já associado no tenant |
| `422` | Body inválido |

A resposta de sucesso contém a configuração persistida. Repetir o mesmo `PUT` na mesma URL com o mesmo body produz o mesmo estado final.

## Alterar parcialmente uma configuração

```http
PATCH /fusionapi/webphone/configs/:userId
```

Envie pelo menos um destes campos: `extension`, `active` ou `uiConfig`.

Exemplo — desativar o Webphone:

```bash
curl --request PATCH \
  --url '<FUSION_API_URL>/fusionapi/webphone/configs/crm-user-123' \
  --header 'Authorization: Bearer <API_KEY>' \
  --header 'Content-Type: application/json' \
  --data '{ "active": false }'
```

Exemplo — alterar configurações de interface:

```bash
curl --request PATCH \
  --url '<FUSION_API_URL>/fusionapi/webphone/configs/crm-user-123' \
  --header 'Authorization: Bearer <API_KEY>' \
  --header 'Content-Type: application/json' \
  --data '{
    "uiConfig": {
      "showDialpad": true,
      "showMute": true,
      "startAsButton": true
    }
  }'
```

> [!IMPORTANT]
> `PATCH` preserva campos de primeiro nível que não foram enviados. Porém, quando `uiConfig` é enviado, ele substitui o objeto `uiConfig` anterior por inteiro. Inclua todas as opções que devem continuar configuradas.

A resposta `200` contém a configuração atualizada.

## Trocar o `userId`

```http
POST /fusionapi/webphone/configs/:userId/change-id
```

Use este endpoint quando o identificador do usuário mudar no sistema integrado.

```bash
curl --request POST \
  --url '<FUSION_API_URL>/fusionapi/webphone/configs/crm-user-123/change-id' \
  --header 'Authorization: Bearer <API_KEY>' \
  --header 'Content-Type: application/json' \
  --data '{ "newUserId": "crm-user-456" }'
```

A resposta `200` contém a configuração com o novo `userId`.

| Status | Resultado |
| ---: | --- |
| `200` | Identificador alterado |
| `404` | `userId` atual não encontrado |
| `409` | `newUserId` já existe no tenant |
| `422` | Body inválido |

Não repita automaticamente a chamada após perder a resposta: se a primeira tentativa tiver sido aplicada, o identificador antigo deixará de existir e a repetição retornará `404`. Consulte primeiro o recurso pelo `newUserId`.

## Formato de erro

Os endpoints públicos de configuração usam este formato:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Payload inválido.",
    "details": [
      "active deve ser boolean."
    ]
  }
}
```

`details` aparece somente quando existem informações adicionais de validação.

| HTTP | `error.code` | Quando ocorre |
| ---: | --- | --- |
| `401` | `INVALID_API_KEY` | API Key ausente ou inválida |
| `404` | `WEBPHONE_NOT_FOUND` | Configuração não encontrada no tenant |
| `404` | `EXTENSION_NOT_FOUND` | Ramal não encontrado no tenant |
| `409` | `WEBPHONE_CONFLICT` | `userId` ou ramal já associado |
| `409` | `USER_ID_ALREADY_EXISTS` | Novo `userId` já está em uso |
| `422` | `VALIDATION_ERROR` | Parâmetro, tipo ou propriedade inválida |
| `500` | `INTERNAL_ERROR` | Falha interna sem detalhes sensíveis |

## Regras de validação

- Objetos com propriedades não documentadas são rejeitados.
- `active` e as opções booleanas de `uiConfig` precisam ser JSON boolean (`true`/`false`), não strings.
- `PUT` requer `extension`, `active` e `uiConfig`.
- `PATCH` rejeita um objeto vazio.
- `uiConfig` precisa ser um objeto, mesmo quando estiver vazio.
- `userId`, `newUserId` e `extension` são normalizados removendo espaços no início e no fim.
- Strings vazias, caracteres de controle e valores acima dos limites documentados são rejeitados.

## Resumo

| Método | Endpoint | Sucesso |
| --- | --- | ---: |
| `GET` | `/fusionapi/webphone/configs` | `200` |
| `GET` | `/fusionapi/webphone/configs/:userId` | `200` |
| `PUT` | `/fusionapi/webphone/configs/:userId` | `200` ou `201` |
| `PATCH` | `/fusionapi/webphone/configs/:userId` | `200` |
| `POST` | `/fusionapi/webphone/configs/:userId/change-id` | `200` |

## Próximos passos

- [Autenticação](Autenticação.md)
- [Integração](Integração.md)
- [Solução de problemas](Troubleshooting.md)
