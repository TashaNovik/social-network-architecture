# Сущности, их поля и связи в микросервисной архитектуре социальной сети (строгое описание)

## 1. Auth Service (PostgreSQL)
### Сущность: auth_credentials
| Поле           | Тип           | Описание                        |
|----------------|---------------|---------------------------------|
| id             | UUID          | Primary Key                     |
| user_id        | UUID          | Связь с User Service            |
| email          | VARCHAR(255)  | Уникальный email                |
| password_hash  | VARCHAR(255)  | Хеш пароля (bcrypt)             |
| created_at     | TIMESTAMP     | Дата создания                   |
| last_login     | TIMESTAMP     | Последний вход                  |
| deleted_at     | TIMESTAMP     | Soft delete                     |

**Индексы:**
- UNIQUE(email) WHERE deleted_at IS NULL
- INDEX(user_id)
- INDEX(deleted_at)

**Связи:**
- user_id → users.id (User Service, 1:1, ON DELETE CASCADE)

---

## 2. User Service (PostgreSQL)
### Сущность: users
| Поле         | Тип           | Описание                        |
|--------------|---------------|---------------------------------|
| id           | UUID          | Primary Key                     |
| first_name   | VARCHAR(100)  | Имя пользователя                |
| last_name    | VARCHAR(100)  | Фамилия пользователя            |
| contact_info | JSONB         | Контактные данные (гибкая структура) |
| created_at   | TIMESTAMP     | Дата создания                   |
| updated_at   | TIMESTAMP     | Дата обновления                 |
| deleted_at   | TIMESTAMP     | Soft delete                     |

**Индексы:**
- INDEX(created_at)
- GIN INDEX(contact_info)
- INDEX(deleted_at)

---

## 3. Post Service (MongoDB)
### Коллекция: posts
| Поле         | Тип           | Описание                        |
|--------------|---------------|---------------------------------|
| _id          | ObjectId      | Primary Key                     |
| user_id      | UUID          | Связь с User Service            |
| content      | Object        | text, media, tags               |
| metadata     | Object        | likes_count, shares_count, visibility |
| created_at   | ISODate       | Дата создания                   |
| updated_at   | ISODate       | Дата обновления                 |
| deleted_at   | ISODate/null  | Soft delete                     |

**Индексы:**
- { user_id, created_at: -1 }
- { created_at: -1 }
- { content.tags: 1 }
- { deleted_at: 1 }

**Связи:**
- user_id → users.id (User Service, N:1, через UUID, без FK)

---

## 4. Comment Service (MongoDB)
### Коллекция: comments
| Поле         | Тип           | Описание                        |
|--------------|---------------|---------------------------------|
| _id          | ObjectId      | Primary Key                     |
| post_id      | UUID          | Связь с Post Service            |
| user_id      | UUID          | Связь с User Service            |
| content      | Object        | text, mentions, reply_to        |
| metadata     | Object        | likes_count, edited             |
| parent_id    | ObjectId/null | Self-reference (threading)      |
| created_at   | ISODate       | Дата создания                   |
| updated_at   | ISODate       | Дата обновления                 |
| deleted_at   | ISODate/null  | Soft delete                     |

**Индексы:**
- { post_id, created_at: 1 }
- { user_id, created_at: -1 }
- { parent_id: 1 }

**Связи:**
- post_id → posts._id (Post Service, N:1, через UUID, без FK)
- user_id → users.id (User Service, N:1, через UUID, без FK)
- parent_id → comments._id (self, threading)

---

## 5. Messaging Service (Cassandra)
### Таблица: messages
| Поле         | Тип           | Описание                        |
|--------------|---------------|---------------------------------|
| chat_id      | UUID          | Primary Key (partition)         |
| message_id   | TIMEUUID      | Primary Key (clustering)        |
| sender_id    | UUID          | Связь с User Service            |
| receiver_id  | UUID          | Связь с User Service            |
| content      | TEXT          | Текст сообщения                 |
| message_type | VARCHAR       | Тип сообщения                   |
| metadata     | MAP<TEXT,TEXT>| Доп. данные                     |
| created_at   | TIMESTAMP     | Дата создания                   |
| read_at      | TIMESTAMP     | Дата прочтения                  |
| deleted_at   | TIMESTAMP     | Soft delete                     |

### Таблица: user_conversations
| Поле             | Тип       | Описание                        |
|------------------|-----------|---------------------------------|
| user_id          | UUID      | Primary Key (partition)         |
| last_message_time| TIMESTAMP | Primary Key (clustering)        |
| chat_id          | UUID      | Идентификатор чата              |
| unread_count     | INT       | Количество непрочитанных        |

**Связи:**
- sender_id, receiver_id → users.id (User Service, N:1, через UUID, без FK)

---

## 6. Notification Service (MongoDB)
### Коллекция: notifications
| Поле         | Тип           | Описание                        |
|--------------|---------------|---------------------------------|
| _id          | ObjectId      | Primary Key                     |
| user_id      | UUID          | Связь с User Service            |
| type         | String        | Тип уведомления                 |
| title        | String        | Заголовок                       |
| content      | Object        | Детали уведомления              |
| channels     | Array         | Каналы доставки                 |
| read         | Boolean       | Прочитано                       |
| read_at      | ISODate/null  | Дата прочтения                  |
| created_at   | ISODate       | Дата создания                   |
| expires_at   | ISODate/null  | Время жизни                     |
| deleted_at   | ISODate/null  | Soft delete                     |

**Индексы:**
- { user_id, created_at: -1 }
- { user_id, read: 1, created_at: -1 }
- { expires_at: 1 }

**Связи:**
- user_id → users.id (User Service, N:1, через UUID, без FK)

---

## 7. Friendship Service (PostgreSQL)
### Сущность: friendships
| Поле         | Тип           | Описание                        |
|--------------|---------------|---------------------------------|
| id           | UUID          | Primary Key                     |
| user_id      | UUID          | Связь с User Service            |
| friend_id    | UUID          | Связь с User Service            |
| status       | VARCHAR(50)   | Статус (друг, подписчик, ...)   |
| created_at   | TIMESTAMP     | Дата создания                   |
| updated_at   | TIMESTAMP     | Дата обновления                 |
| deleted_at   | TIMESTAMP     | Soft delete                     |

**Индексы:**
- UNIQUE(user_id, friend_id)
- INDEX(status)
- INDEX(deleted_at)

**Связи:**
- user_id, friend_id → users.id (User Service, N:M, ON DELETE CASCADE)

---

## 8. Media Service (S3 + PostgreSQL)
### Сущность: media
| Поле         | Тип           | Описание                        |
|--------------|---------------|---------------------------------|
| id           | UUID          | Primary Key                     |
| user_id      | UUID          | Связь с User Service            |
| file_path    | VARCHAR(255)  | Путь к файлу в S3               |
| file_type    | VARCHAR(50)   | Тип файла                       |
| metadata     | JSONB         | Метаданные                      |
| created_at   | TIMESTAMP     | Дата загрузки                   |
| deleted_at   | TIMESTAMP     | Soft delete                     |

**Индексы:**
- UNIQUE(file_path)
- INDEX(user_id)
- INDEX(deleted_at)

**Связи:**
- user_id → users.id (User Service, N:1, ON DELETE CASCADE)
- file_path — путь к объекту в S3 (файл вне БД)

---

## 9. Search Service (Elasticsearch)
### Индексы
- users: поиск по имени, email
- posts: поиск по тексту, тегам
- comments: поиск по тексту
- messages: поиск по тексту

**Связи:**
- Индексация данных из других сервисов через события (Kafka), без прямых FK

---

## 10. Admin Service (PostgreSQL)
### Сущность: admin_actions
| Поле         | Тип           | Описание                        |
|--------------|---------------|---------------------------------|
| id           | UUID          | Primary Key                     |
| admin_id     | UUID          | Связь с User Service            |
| action_type  | VARCHAR(50)   | Тип действия                    |
| target_id    | UUID          | ID целевого объекта             |
| created_at   | TIMESTAMP     | Дата действия                   |
| details      | JSONB         | Детали действия                 |

**Индексы:**
- INDEX(admin_id)
- INDEX(target_id)
- INDEX(created_at)

**Связи:**
- admin_id → users.id (User Service, N:1, ON DELETE SET NULL)
- target_id — логическая связь с любым объектом (user, post, comment и др.)

---

## 11. NewsFeed Service (PostgreSQL)
### Сущность: newsfeed
| Поле         | Тип           | Описание                        |
|--------------|---------------|---------------------------------|
| id           | UUID          | Primary Key                     |
| user_id      | UUID          | Связь с User Service            |
| post_id      | UUID          | Связь с Post Service            |
| created_at   | TIMESTAMP     | Дата добавления                 |
| updated_at   | TIMESTAMP     | Дата обновления                 |
| deleted_at   | TIMESTAMP     | Soft delete                     |

**Индексы:**
- UNIQUE(user_id, post_id)
- INDEX(user_id)
- INDEX(deleted_at)

**Связи:**
- user_id → users.id (User Service, N:1, ON DELETE CASCADE)
- post_id — логическая связь с posts._id (Post Service, N:1, через UUID, без FK)

---

## 12. Общие паттерны и особенности
- Везде реализован soft delete через поле deleted_at.
- Все идентификаторы — UUID (или ObjectId для MongoDB).
- Все связи межсервисные, реализуются через идентификаторы (UUID/ObjectId), без жёстких FK между сервисами.
- Для MongoDB и Cassandra — связи только на уровне приложения (reference by UUID/ObjectId).
- Для PostgreSQL — FK только внутри сервиса, межсервисные — через UUID.
- Все реляционные таблицы нормализованы до 3NF, для гибких структур используются JSONB/JSON.
- Индексация и поиск — через GIN индексы (PostgreSQL), TTL индексы (MongoDB), партиционирование (Cassandra), Elasticsearch.
- Масштабируемость: партиционирование, шардинг, read replicas, кэширование (Redis).
- Eventual consistency: межсервисные связи поддерживаются через события (Kafka), без жёстких FK.

---

## 13. Примеры полезных структур и сценариев

### Пример структуры contact_info (User Service)
```json
{
  "phone": "+1234567890",
  "telegram": "@username",
  "email_public": "user@example.com",
  "website": "https://example.com"
}
```

### Пример структуры content (Post Service)
```json
{
  "text": "My first post!",
  "media": ["s3://bucket/file1.jpg"],
  "tags": ["#firstpost"]
}
```

### Пример структуры metadata (Comment Service)
```json
{
  "likes_count": 15,
  "edited": false
}
```

