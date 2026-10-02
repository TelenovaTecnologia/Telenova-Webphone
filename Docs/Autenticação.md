# Autenticação

A integração utiliza duas credenciais:

- **API Key do tenant** — usada na API pública de configuração e no runtime do Webphone;
- **JWT administrativo** — usado para consultar a API Key do tenant.

## Fluxo completo

```text
Credenciais do painel
        │
        ▼
POST /fusionapi/auth
        │ retorna JWT
        ▼
GET /fusionapi/webphone/integration-key
        │ retorna API Key
        ▼
Configurar usuários e carregar o Webphone
```

Se a sua equipe já recebeu a API Key do tenant, não é necessário repetir o login administrativo em cada carregamento. Armazene a chave conforme a política de segurança da integração.

## 1. Obter o JWT administrativo

```http
POST <FUSION_API_URL>/fusionapi/auth
Content-Type: application/json

{
  "username": "usuario",
  "password": "senha",
  "domain": "empresa.exemplo.com"
}
```

Todos os campos são obrigatórios. `domain` é o nome do domínio cadastrado no painel, não uma URL e não o `domainUuid`.

Resposta de sucesso:

```json
{
  "user": {
    "user_uuid": "uuid-do-usuario",
    "username": "usuario"
  },
  "token": "<JWT>",
  "expireAt": "2026-10-03T12:00:00.000Z"
}
```

O objeto `user` pode conter outros dados do perfil. Para este fluxo, use apenas `token` e `expireAt`. O JWT atual expira em 1 dia.

| Status | Significado |
| ---: | --- |
| `200` | Login realizado |
| `400` | Algum campo obrigatório não foi enviado |
| `401` | Usuário, senha ou domínio inválido |
| `403` | Domínio desabilitado |

## 2. Consultar a API Key do tenant

```http
GET <FUSION_API_URL>/fusionapi/webphone/integration-key
Authorization: Bearer <JWT>
```

Resposta quando a chave existe:

```json
{
  "apiKey": "wp_tk_EXEMPLO"
}
```

Resposta quando o tenant ainda não possui chave:

```json
{
  "apiKey": null
}
```

O tenant é obtido do usuário autenticado no JWT. Não envie `domainUuid` na query ou no body.

> [!NOTE]
> Esta documentação cobre apenas a consulta da chave. A criação automática de uma nova API Key ainda não faz parte do contrato disponível no ambiente de homologação.

## 3. Usar a API Key

Nos endpoints da Fusion API, envie:

```http
Authorization: Bearer <API_KEY>
```

Exemplo:

```http
GET <FUSION_API_URL>/fusionapi/webphone/configs
Authorization: Bearer wp_tk_EXEMPLO
```

No carregamento por tag `<script>` ou iframe, a chave é enviada no parâmetro `token`, pois esses elementos não permitem configurar o header `Authorization`:

```text
<WEBPHONE_URL>/api/webphone?userId=<USER_ID>&token=<API_KEY>
<WEBPHONE_URL>/embed?userId=<USER_ID>&token=<API_KEY>
```

## Escopo da API Key

A API Key identifica um tenant. Com ela é possível:

- consultar e alterar as configurações de Webphone daquele tenant;
- carregar qualquer Webphone ativo daquele tenant quando o `userId` for conhecido.

Ela não permite consultar configurações de outro tenant. O domínio é sempre derivado da própria chave.

## Cuidados com a API Key

No modelo atual, a mesma API Key é usada na configuração e chega ao navegador durante o runtime. Portanto:

- nunca versione uma chave real no Git;
- nunca inclua a chave em logs, analytics, mensagens de erro, prints ou tickets;
- use somente HTTPS;
- evite salvar a URL completa do script ou iframe em ferramentas de observabilidade;
- não armazene a chave em `localStorage` ou `sessionStorage` por iniciativa da aplicação hospedeira;
- ao compartilhar exemplos, substitua a chave por `wp_tk_EXEMPLO`;
- se houver suspeita de exposição, interrompa o uso da chave e solicite sua substituição pelos responsáveis pelo ambiente.

## Erros comuns

| Sintoma | Causa provável | Ação |
| --- | --- | --- |
| `401` na Fusion API | API Key ou JWT ausente/inválido | Confira o tipo de token usado no endpoint |
| `403` no runtime | Chave ausente, tenant incorreto ou Webphone inativo | Confira chave, `userId` e campo `active` |
| `apiKey: null` | Tenant ainda não possui chave | Solicite a criação da chave no ambiente responsável |
| Login retorna `401` | Credenciais ou domínio incorretos | Teste o mesmo acesso no painel |

## Próximos passos

- [Referência da API](API.md)
- [Integração](Integração.md)
- [Solução de problemas](Troubleshooting.md)
