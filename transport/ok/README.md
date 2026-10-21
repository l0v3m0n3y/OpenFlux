# transport/ok — OpenFlux carrier через звонки ok.ru (PORTAL)

Каркас по образцу `transport/oneme`. Медиа-слой (videowebrtc + ICE injection)
почти тот же; **отличается только auth + start/accept call**.

## Соответствие файлов

| oneme | ok | Назначение |
|-------|-----|------------|
| max_transport.go | ok_transport.go | Start/Stop/Send, роль exit/client |
| max_wclient.go | ok_wclient.go | Сессия ok.ru (cookies), HTTP API |
| max_call.go | ok_call.go | WebRTC + ws2 videowebrtc |
| max_call_helper.go | ok_call_helper.go | craftEndpoint с clientType=PORTAL |
| max_data.go | ok_data.go | типы |
| max_utils.go | ok_utils.go | log/uuid |

## Поток

```
Exit:  cookies A → (TODO listen incoming) → ws2 PORTAL tgt=accept → DC/ICE tunnel
Client: cookies B → getCallToken/start → ws2 PORTAL tgt=start → uid A → tunnel
```

## Флаги (предложение)

```
--transport=ok
--ok-cookies=<string|path>   # Cookie: AUTHCODE=...; JSESSIONID=...
--ok-uid=<id>                # userId exit (client звонит сюда), напр. 910703757272
```

## getCallToken (снято)

```
POST https://ok.ru/web-api/v2/videochat/getCallToken
body: {"id": 30}
→ {
    "success": true,
    "result": {
      "apiUrl": "https://calls.okcdn.ru",
      "token":  "$0tOAysM...QDYC5aJMwr"
    }
  }
```

`id: 30` — не peer uid. Token — для **calls.okcdn.ru**, не для ws2.

## Следующий HAR (шаг 2)

Фильтр Network: `calls.okcdn.ru` сразу после getCallToken при **исходящем** звонке.

Нужны: URL, method, headers (Authorization?), body, response → conversationId / media token / videowebrtc.

Плюс: как exit узнаёт о **входящем** звонке.


## Шаг 2 — calls.okcdn.ru/fb.do (снято)

POST https://calls.okcdn.ru/fb.do
  method=auth.anonymLogin
  application_key=CPCCMQKGDIHBABABA
  session_data={version, device_id, client_type:SDK_JS, auth_token:$...}

→ uid, session_key, api_server

Media/TURN response (method name TBD):
  token, endpoint wss://videowebrtc.okcdn.ru/ws2, turn_server, client_type PORTAL, external_user_type ODKL

## Known calls.okcdn.ru methods

| method | params | result |
|--------|--------|--------|
| auth.anonymLogin | session_data, application_key | uid, session_key |
| system.getInfo | session_key | serverTime |
| vchat.getConversationParams | session_key, conversationId | token, endpoint, turn, PORTAL |

Still TODO: method that **creates** conversationId / starts call to peer uid.
