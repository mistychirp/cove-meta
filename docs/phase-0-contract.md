# Cove — Phase 0 Contract (draft v0.1)

> Это **контракт**, а не реализация. Референсный сервер и клиент — просто первые
> его реализации. Стабильность и версионирование этого документа — то, что
> отличает Cove от Discord / Stoat / Spacebar и делает возможным "море клиентов".

## 0. Сквозные решения

| Тема | Решение | Почему |
|------|---------|--------|
| Идентификаторы | **UUIDv7** (текстом в API) | Сортируемы по времени, стандарт, без координации worker-id как у Snowflake. Дают бесплатную пагинацию по id. |
| Время | UTC, RFC 3339 строкой (`2026-07-14T19:29:00Z`) | Однозначно, человекочитаемо. |
| Пустые списки | Всегда `[]`, **никогда `null`** | Клиент не должен защищаться от двух форм пустоты. (В Go nil-слайс сериализуется в `null` — инициализируем через `make(...)`.) |
| Транспорт событий | WebSocket gateway, envelope `{op, t, s, d}` | Как у Discord, но задокументированный и версионированный. |
| REST-версия | префикс пути `/api/v1/...` | Явное версионирование контракта. |
| Gateway-версия | query `?v=1` при подключении | Клиент и сервер могут расходиться в версиях — self-host требует этого. |
| Права | 64-битная маска, роли + оверрайды на канал | Модель Discord, проверенная годами. |
| Контент сообщения | сырой markdown-строкой, рендер на клиенте | Проще хранить/искать; entity-модель Telegram избыточна для старта. |

---

## 1. Модель данных (Phase 0)

Поля со `*` — обязательны в Phase 0. Остальные заложены на будущее, но таблицы
создаём сразу, чтобы не мигрировать больно.

### User
```
id*            uuidv7 (pk)
username*      text, unique, [a-z0-9_], 2..32
display_name*  text, 1..64
email*         text, unique (для входа/восстановления)
password_hash* text (argon2id)
avatar         text (media id, nullable)
instance_owner* bool, default false   # может выпускать RegistrationToken (см. §6)
created_at*    timestamptz
```

### Session
```
id*         uuidv7 (pk)
user_id*    uuidv7 (fk User)
token_hash* bytea, unique        # sha256(bearer-токен); сам токен не хранится
created_at* timestamptz
expires_at* timestamptz
user_agent  text
```

### Server  (в API — "server"; внутри допустимо "guild")
```
id*         uuidv7 (pk)
name*       text, 1..100
owner_id*   uuidv7 (fk User)
icon        text (media id, nullable)
created_at* timestamptz
```

### Member  (членство пользователя в сервере)
```
server_id*  uuidv7 (fk Server)   # (server_id, user_id) = pk
user_id*    uuidv7 (fk User)
nickname    text, nullable
joined_at*  timestamptz
# роли участника — через таблицу MemberRole (many-to-many)
```

### Role
```
id*          uuidv7 (pk)
server_id*   uuidv7 (fk Server)
name*        text, 1..100
color        int (0xRRGGBB, nullable)
permissions* bigint (битовая маска, см. §2)
position*    int (иерархия; больше = выше)
hoist*       bool (показывать отдельной группой в списке)
mentionable* bool
```
> В каждом сервере есть неудаляемая роль `@everyone` с `position = 0`.

### Channel
```
id*         uuidv7 (pk)
server_id   uuidv7 (fk Server, nullable)   # null => DM/групповой чат (Phase 1)
type*       enum: category | text | voice | dm
name*       text, 1..100
topic       text, nullable
position*   int
parent_id   uuidv7 (fk Channel, nullable)   # для вложенности в категорию
created_at* timestamptz
```

### ChannelRecipient  (участники личного чата)
```
channel_id*  uuidv7 (fk Channel)   # (channel_id, user_id) = pk
user_id*     uuidv7 (fk User)
```
> Личный чат = канал с `server_id = null` и `type = 'dm'`. Участников негде было
> хранить (`Member` привязан к серверу), поэтому отдельная таблица. Права в DM
> дают не роли, а само участие: `VIEW_CHANNEL | SEND_MESSAGES | ATTACH_FILES`.

### ChannelOverride  (пер-канальные оверрайды прав)
```
channel_id*   uuidv7 (fk Channel)
target_type*  enum: role | member
target_id*    uuidv7                # role_id или user_id
allow*        bigint (маска)
deny*         bigint (маска)
```
> В Phase 0 можно не давать UI для оверрайдов, но резолвер прав их уже учитывает.

### Message
```
id*         uuidv7 (pk)            # сортировка = порядок сообщений
channel_id* uuidv7 (fk Channel)
author_id*  uuidv7 (fk User)
content*    text (markdown), 0..4000
type*       enum: default | system
reply_to    uuidv7 (fk Message, nullable)
created_at* timestamptz
edited_at   timestamptz, nullable
```

### Attachment
```
id*          uuidv7 (pk)
message_id*  uuidv7 (fk Message)      # вложение всегда принадлежит сообщению
media_id*    text (ключ в storage-слое)
filename*    text
content_type text
size         bigint
```
> Хранилище — за интерфейсом (`internal/storage`): дефолт `local` (диск,
> `COVE_STORAGE_PATH`), S3-совместимое добавляется той же реализацией интерфейса.
> В JSON вложение отдаётся с относительным `url` (`/api/v1/attachments/<id>`) —
> клиент подставляет адрес своего инстанса.

### Invite  (вступление в конкретный СЕРВЕР)
```
code*        text (pk, короткий, напр. base62 8 симв.)
server_id*   uuidv7 (fk Server)
created_by*  uuidv7 (fk User)
max_uses     int, nullable (null = бесконечно)
uses*        int, default 0
expires_at   timestamptz, nullable
created_at*  timestamptz
```

### RegistrationToken  (доступ к ИНСТАНСУ — модель Matrix)
```
code*        text (pk, короткий)
created_by*  uuidv7 (fk User)          # админ инстанса
max_uses     int, nullable
uses*        int, default 0
expires_at   timestamptz, nullable
created_at*  timestamptz
```
> Принципиально отдельная сущность от `Invite`. `RegistrationToken` открывает
> регистрацию аккаунта **на инстансе**; `Invite` добавляет уже существующего
> пользователя **в сервер**. Регистрация «открыта», но требует валидного токена,
> выданного владельцем инстанса.

---

## 2. Модель прав

64-битная маска. Стартовый набор (номер = сдвиг бита):

| Бит | Право | Смысл |
|-----|-------|-------|
| 0 | `VIEW_CHANNEL` | видеть канал и читать историю |
| 1 | `SEND_MESSAGES` | писать сообщения |
| 2 | `MANAGE_MESSAGES` | удалять/закреплять чужие сообщения |
| 3 | `MANAGE_CHANNELS` | создавать/менять/удалять каналы |
| 4 | `MANAGE_ROLES` | управлять ролями (не выше своей) |
| 5 | `KICK_MEMBERS` | кикать |
| 6 | `BAN_MEMBERS` | банить |
| 7 | `MANAGE_SERVER` | имя/иконка/настройки сервера |
| 8 | `CREATE_INVITE` | создавать инвайты |
| 9 | `MENTION_EVERYONE` | пинговать @everyone |
| 10 | `ATTACH_FILES` | прикреплять файлы |
| 11 | `CONNECT` | зайти в голосовой канал |
| 12 | `SPEAK` | говорить и демонстрировать экран |
| 63 | `ADMINISTRATOR` | обходит все проверки |

**Порядок резолва прав участника в канале:**
1. Владелец сервера ИЛИ бит `ADMINISTRATOR` → всё разрешено, стоп.
2. `base = permissions` роли `@everyone`.
3. `base |= permissions` каждой роли участника (OR).
4. Применить оверрайд `@everyone` канала: `base = (base & ~deny) | allow`.
5. Собрать оверрайды ролей участника: `base = (base & ~Σdeny) | Σallow`.
6. Применить member-оверрайд канала (высший приоритет).

---

## 3. Gateway-протокол (WebSocket, v1)

Подключение: `GET /gateway?v=1&encoding=json` → апгрейд до WebSocket.

**Envelope каждого фрейма:**
```jsonc
{
  "op": 0,          // опкод (см. ниже)
  "t":  "MESSAGE_CREATE", // тип события, только для op=0 (DISPATCH)
  "s":  42,         // sequence, только для DISPATCH; клиент шлёт в heartbeat
  "d":  { }         // полезная нагрузка
}
```

**Опкоды:**
| op | Имя | Направление | Назначение |
|----|-----|-------------|------------|
| 0 | DISPATCH | S→C | доставка события (`t` + `d`) |
| 1 | HELLO | S→C | первый фрейм: `{ heartbeat_interval_ms }` |
| 2 | IDENTIFY | C→S | `{ token }` |
| 3 | READY | S→C | начальное состояние (см. ниже) |
| 4 | HEARTBEAT | C→S | `{ s }` (последний полученный sequence) |
| 5 | HEARTBEAT_ACK | S→C | подтверждение |
| 6 | RESUME | C→S | `{ token, session_id, s }` — переподключение без потери |
| 9 | INVALID_SESSION | S→C | нужно переидентифицироваться заново |

**Хендшейк:** connect → `HELLO` → клиент шлёт `IDENTIFY` → сервер шлёт `READY`
→ далее клиент шлёт `HEARTBEAT` каждые `heartbeat_interval_ms`.

**`READY.d`** (начальный снапшот):
```jsonc
{
  "session_id": "…",
  "user": { /* self */ },
  "servers": [ { "id", "name", "icon",
                 "channels": [...], "roles": [...],
                 "members_count": 7 } ],
  "dms": [ { "id", "type": "dm", "server_id": null,
             "recipients": [ /* User */ ] } ]
}
```

**События DISPATCH (Phase 0):**
```
MESSAGE_CREATE   MESSAGE_UPDATE   MESSAGE_DELETE
CHANNEL_CREATE   CHANNEL_UPDATE   CHANNEL_DELETE
ROLE_CREATE      ROLE_UPDATE      ROLE_DELETE
MEMBER_JOIN      MEMBER_UPDATE    MEMBER_LEAVE
TYPING_START     PRESENCE_UPDATE  DM_CREATE
```
> События канала доставляются только тем, у кого есть `VIEW_CHANNEL`
> (для личного чата — только его участникам). Подписка: каналы сервера — через
> сервер, личные чаты — поканально.
`d` каждого события = соответствующий объект из §1 (или его дельта).
`TYPING_START.d` = `{ channel_id, user_id, user }` — юзер вложен целиком,
чтобы клиенту не приходилось отдельно резолвить имя.

---

## 4. REST-поверхность (Phase 0)

Все под `/api/v1`. Аутентификация — заголовок `Authorization: <token>`.

```
# auth
POST   /auth/register            { username, display_name, email, password, registration_token }
POST   /auth/login               { login, password } -> { token, user }
POST   /auth/logout

# self
GET    /users/@me
PATCH  /users/@me
GET    /users/@me/channels          # личные чаты
POST   /users/@me/channels          { username } | { recipient_id }
                                    # создаёт чат или возвращает существующий (200)

# servers
POST   /servers                  { name } -> Server
GET    /servers/:id
PATCH  /servers/:id
DELETE /servers/:id
GET    /servers/:id/members
GET    /servers/:id/channels
POST   /servers/:id/channels     { type, name, parent_id? }
GET    /servers/:id/roles
POST   /servers/:id/roles

# channels
GET    /channels/:id
PATCH  /channels/:id
DELETE /channels/:id
GET    /channels/:id/messages    ?before=<msgid>&limit=50   (курсорная пагинация)
POST   /channels/:id/messages    { content, reply_to? }
       # либо multipart/form-data: поле payload_json (тот же JSON) + поля files.
       # Файлы отправляются вместе с сообщением одним запросом — вложений-сирот
       # не бывает by design. Требует права ATTACH_FILES.

GET    /attachments/:id          # отдаёт файл; проверяет VIEW_CHANNEL канала сообщения
PATCH  /channels/:id/messages/:mid
DELETE /channels/:id/messages/:mid
POST   /channels/:id/typing
POST   /channels/:id/voice/token    # -> { url, token, can_speak }
       # Токен LiveKit для голосового канала. Комната = id канала,
       # личность = id юзера. Нужен CONNECT; без SPEAK — can_speak:false
       # (заходит слушателем, canPublish=false прямо в токене).
       # 503, если на инстансе голос не настроен.

# invites
POST   /servers/:id/invites      { max_uses?, expires_at? } -> Invite
POST   /invites/:code            # принять инвайт, вступить в сервер
GET    /invites/:code            # превью сервера до вступления

# admin (инстанс; требует instance_owner)
POST   /admin/registration-tokens   { max_uses?, expires_at? } -> RegistrationToken
GET    /admin/registration-tokens
DELETE /admin/registration-tokens/:code

# инстанс (публично) — клиент должен знать возможности конкретного сервера
GET    /instance                 -> { voice_enabled, open_registration, max_upload_mb }

# gateway discovery
GET    /gateway                  -> { url: "wss://host/gateway" }
```

---

## 5. Зафиксированные решения

1. **Бэкенд** — Go + PostgreSQL + sqlc. ✓
2. **Регистрация** — открытая, но по `RegistrationToken` от владельца инстанса (модель Matrix). ✓
3. **Presence** — online/offline в Phase 0, выводится из состояния gateway-подключений. ✓
4. **Репозитории** — корневой мета-репо (`docs/`) + независимые `cove-server/` и `cove-web/`. ✓
5. **Клиент** — React (Vite). ✓
6. **Dev-окружение** — Nix flake dev-shell в `cove-server/` (go, sqlc, golang-migrate, psql). ✓

## 6. Роль «владельца инстанса»

Первый зарегистрированный аккаунт (либо заданный через `COVE_ADMIN_USERNAME`)
получает `instance_owner = true`. Только он выпускает `RegistrationToken` и позже
управляет инстансом. Это **не** то же самое, что владелец сервера
(`Server.owner_id`): один — про инстанс, другой — про конкретный сервер.
