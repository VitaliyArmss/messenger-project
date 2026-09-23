# Security

## 1. Security model

Основная security-модель проекта строится в несколько уровней:

```text
Authentication
     ↓
JWT validation
     ↓
Endpoint protection
     ↓
Application-level authorization
     ↓
Data access rules
```

То есть наличие валидного JWT само по себе не означает доступ ко всем данным.

---

## 2. Authentication

API использует JWT Bearer authentication.

Backend проверяет:

- issuer;
- audience;
- lifetime;
- signing key.

Подробный lifecycle access/refresh tokens находится в [authentication.md](./authentication.md).

---

## 3. Protected endpoints

SignalR hubs используют `[Authorize]`.

Это относится к:

```text
ChatHub
NotificationHub
```

REST endpoints также работают с текущим user identity для выполнения пользовательских операций.

---

## 4. Authorization внутри чатов

После authentication backend выполняет application-level checks.

Например, при работе с chat data проверяется membership пользователя:

```text
User
  ↓
ChatMember
  ↓
Chat
```

Если пользователь не состоит в чате, операции, требующие membership, должны завершаться отказом в доступе.

---

## 5. Group owner permissions

В группах существует owner.

Это используется для операций, которые должны быть доступны только владельцу:

- удаление группы;
- изменение имени и avatar группы;
- управление участниками в сценариях, где требуется owner permission.

При удалении участника логика позволяет владельцу удалить другого участника, а обычному пользователю — удалить самого себя.

---

## 6. File access control

Attachment не считается публичным только потому, что он существует в MinIO.

При скачивании backend:

1. находит attachment;
2. определяет его связь с chat;
3. проверяет доступ текущего пользователя;
4. только после этого получает object из MinIO.

```text
Request
  ↓
JWT identity
  ↓
Attachment lookup
  ↓
Access check
  ↓
MinIO
```

Это позволяет централизовать access control на уровне приложения.

---

## 7. CORS

CORS policy не является механизмом authentication, но ограничивает browser-origin access к API.

Allowed origins конфигурируются отдельно через `Cors:Origins`.

При раздельном hosting frontend/backend это особенно важно, поскольку приложение работает между разными origin.

---

## 8. Secrets management

Production secrets не должны находиться в Git repository.

К ним относятся:

```text
JWT signing key
Database password
MinIO access key
MinIO secret key
```

В конфигурации проекта предусмотрено получение этих значений через environment/configuration.

---

## 9. Текущие ограничения security

Документация намеренно фиксирует не только реализованные меры, но и существующие ограничения.

### Refresh token на frontend

Текущая React-реализация хранит access и refresh tokens в `localStorage`.

Более строгая production-схема может использовать `HttpOnly` и `Secure` cookies для refresh credentials.

### Rate limiting

Отдельный rate limiting для login, registration и refresh endpoints в текущей реализации не выделен.

### Centralized exception handling

В controllers присутствуют локальные `try/catch`, но централизованный exception-handling middleware не выделен в отдельный слой.

### Automated security testing

Автоматизированных security/integration tests в текущем проекте недостаточно для утверждения о полном покрытии access-control сценариев.

---

## 10. Важное ограничение текущей реализации сообщений

Операции редактирования и удаления сообщения уже существуют и синхронизируются через SignalR, однако в текущем `MessageService` проверка того, что редактирование/удаление выполняет именно автор сообщения либо пользователь с соответствующим правом, не доведена до отдельной authorization rule.

Это следует рассматривать как технический долг перед дальнейшим production hardening.