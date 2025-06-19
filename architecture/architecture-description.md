Диаграмма отражает архитектуру микросервисной социальной платформы, разделённую на домены и уровни взаимодействия. Ниже описаны все микросервисы, их функции, используемые технологии и взаимодействие между ними.

## Общая структура

Диаграмма включает следующие основные компоненты:
- **Клиенты**: User (Пользователь), Administrator (Администратор) — веб/мобильные клиенты.
- **Load Balancer [nginx/HAProxy]**: Балансирует входящий трафик между API Gateway.
- **API Gateway [Kong/Nginx]**: Центральная точка входа для маршрутизации запросов к микросервисам по gRPC/REST.
- **Внешние интеграции**:
  - **External Services [Email, Push]** — для отправки уведомлений.
  - **CDN [CloudFront]** — для доставки медиафайлов пользователям напрямую по HTTPS.
- **Доменные микросервисы**, разделённые на 4 группы:
  - **Core & Media**: Notification Service, Media Service, Messaging Service.
  - **Supporting Services**: Search Service, Admin Service.
  - **Content & Feeds**: Post Service, Comment Service, NewsFeed Service.
  - **User & Social Graph**: Auth Service, User Service, Friendship Service.
- **Shared Infrastructure**: Redis Cluster (Cache, Pub/Sub), Apache Kafka (Event Bus).

## Внешние сущности и интеграции

- **User (Пользователь)**: 
  - Использует API (HTTPS) для взаимодействия с системой через Load Balancer и API Gateway.
  - Получает медиа напрямую через CDN (CloudFront) по HTTPS.
- **Administrator (Администратор)**: 
  - Использует Admin API (HTTPS) через Load Balancer и API Gateway.
- **External Services (Email, Push)**: 
  - Получают уведомления от Notification Service.
- **CDN (CloudFront)**: 
  - Доставляет медиафайлы пользователям.

## Load Balancer

- **Load Balancer [nginx/HAProxy]**:
  - Балансирует входящий трафик между API Gateway.

## API Gateway

- **API Gateway [Kong/Nginx]**:
  - Центральная точка входа для всех внешних запросов.
  - Маршрутизирует запросы к микросервисам по gRPC/REST.

## Доменные микросервисы

### 1. Core & Media

- **Notification Service [MongoDB]**:
  - Функция: Отправляет уведомления (Email, Push) через внешние сервисы.
  - Технологии: MongoDB для хранения данных, Redis Cluster для кэширования и подписки на уведомления (Sub: for Notifications).
  - Взаимодействие: Отправляет уведомления внешним сервисам.

- **Media Service [S3, PostgreSQL]**:
  - Функция: Управляет медиафайлами, доставляет их через CDN (CloudFront).
  - Технологии: S3 для хранения файлов, PostgreSQL для метаданных, Redis для подписки на уведомления (Pub/Sub).
  - Взаимодействие: Доставляет медиа через CDN.

- **Messaging Service [Cassandra, WebSocket]**:
  - Функция: Реализует обмен сообщениями в реальном времени.
  - Технологии: Cassandra для хранения сообщений, WebSocket для обмена данными, Redis для подписки на уведомления (Pub/Sub).
  - Взаимодействие: Обрабатывает сообщения пользователей через WebSocket.

### 2. Supporting Services

- **Search Service [Elasticsearch]**:
  - Функция: Поиск по данным системы.
  - Технологии: Elasticsearch для индексации и поиска, Apache Kafka для подписки на поисковые события (Sub: for Search).
  - Взаимодействие: Получает события поиска через Kafka.

- **Admin Service [PostgreSQL]**:
  - Функция: Управление административными задачами.
  - Технологии: PostgreSQL для хранения данных.
  - Взаимодействие: Принимает запросы от API Gateway по gRPC/REST.

### 3. Content & Feeds

- **Post Service [MongoDB]**:
  - Функция: Управляет постами пользователей.
  - Технологии: MongoDB для хранения данных, Apache Kafka для публикации событий контента (Pub: Content Events).
  - Взаимодействие: Принимает запросы через API Gateway (gRPC/REST), публикует события в Kafka.

- **Comment Service [MongoDB]**:
  - Функция: Управляет комментариями к постам.
  - Технологии: MongoDB для хранения данных, Apache Kafka для публикации событий контента (Pub: Content Events).
  - Взаимодействие: Принимает запросы через API Gateway (gRPC/REST), публикует события в Kafka.

- **NewsFeed Service [PostgreSQL]**:
  - Функция: Формирует новостные ленты пользователей.
  - Технологии: PostgreSQL для хранения данных, Apache Kafka для подписки на события лент (Sub: for Feeds) и публикации событий контента (Pub: Content Events).
  - Взаимодействие: Принимает запросы через API Gateway, взаимодействует с Kafka.

### 4. User & Social Graph

- **Auth Service [PostgreSQL]**:
  - Функция: Аутентификация и авторизация.
  - Технологии: PostgreSQL для хранения данных, Apache Kafka для публикации событий пользователей (Pub: User Events).
  - Взаимодействие: Принимает запросы через API Gateway (gRPC/REST), публикует события в Kafka.

- **User Service [PostgreSQL]**:
  - Функция: Управление данными пользователей.
  - Технологии: PostgreSQL для хранения данных, Apache Kafka для публикации событий пользователей (Pub: User Events).
  - Взаимодействие: Принимает запросы через API Gateway (gRPC/REST), публикует события в Kafka.

- **Friendship Service [PostgreSQL]**:
  - Функция: Управление социальными связями (дружбой).
  - Технологии: PostgreSQL для хранения данных, Apache Kafka для публикации событий пользователей (Pub: User Events).
  - Взаимодействие: Принимает запросы через API Gateway (gRPC/REST), публикует события в Kafka.

## Shared Infrastructure

- **Redis Cluster [Cache, Pub/Sub]**:
  - Кэширование данных и механизм публикации/подписки.
  - Используется сервисами: Notification Service (Sub: for Notifications), Media Service, Messaging Service.

- **Apache Kafka [Event Bus]**:
  - Асинхронное взаимодействие между сервисами через шину событий.
  - Поддерживает:
    - Публикацию событий контента (Post Service, Comment Service, NewsFeed Service) — Pub: Content Events.
    - Публикацию событий пользователей (Auth Service, User Service, Friendship Service) — Pub: User Events.
    - Подписку на события лент (NewsFeed Service) — Sub: for Feeds.
    - Подписку на события поиска (Search Service) — Sub: for Search.

## Взаимодействие между компонентами

1. **Пользователи и администраторы** отправляют запросы через Load Balancer и API Gateway по HTTPS.
2. **API Gateway** маршрутизирует запросы к микросервисам по gRPC/REST.
3. **Notification Service** отправляет уведомления через внешние сервисы (Email, Push), используя Redis для кэширования и подписки (Sub: for Notifications).
4. **Media Service** доставляет медиа через CDN (CloudFront), подписывается на уведомления через Redis.
5. **Messaging Service** обеспечивает обмен сообщениями через WebSocket, подписывается на уведомления через Redis.
6. **Search Service** получает поисковые события через Apache Kafka (Sub: for Search).
7. **Admin Service** обрабатывает административные запросы от API Gateway.
8. **Post Service, Comment Service, NewsFeed Service** публикуют события контента в Kafka (Pub: Content Events); NewsFeed Service также подписывается на события лент (Sub: for Feeds).
9. **Auth Service, User Service, Friendship Service** публикуют события пользователей в Kafka (Pub: User Events).
10. **Apache Kafka** служит шиной событий для асинхронного обмена данными.
11. **Redis Cluster** используется для кэширования и Pub/Sub механизма уведомлений.
12. **Пользователи получают медиа напрямую через CDN (CloudFront) по HTTPS.**

Архитектура представляет чётко структурированную систему микросервисов. Каждый сервис отвечает за свою задачу, взаимодействуя через API Gateway для синхронных запросов и Apache Kafka для асинхронных событий. Redis обеспечивает кэширование и уведомления, а CDN (CloudFront) — быструю доставку медиа, что делает систему масштабируемой и надёжной.