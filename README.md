# Декомпозиция и проектирование микросервисной системы: «Ветеринарная клиника»

## Анализ предметной области

### Описание системы
Информационная система сети ветеринарных клиник автоматизирует онлайн-запись питомцев на приём, ведение электронных медицинских карт (история болезней, вакцинация, диагностические снимки), расчёт стоимости услуг с онлайн-предоплатой и автоматические напоминания владельцам. Система обслуживает владельцев животных, ветеринарных врачей и администраторов клиники.
Единой точкой входа для веб- приложения служит API Gateway, а межсервисное синхронное взаимодействие строится на базе gRPC, при этом каждый сервис владеет собственной изолированной базой данных PostgreSQL.

### Акторы
**Пользователи системы:**
- **Владелец питомца (`OWNER`)** — регистрирует питомцев, выбирает врача и услугу, записывается на приём, вносит предоплату, просматривает медкарту и получает уведомления.
- **Ветеринарный врач (`VET`)** — просматривает своё расписание записей, заполняет медицинскую карту по итогам приёма, прикрепляет результаты анализов и снимки (УЗИ/рентген).
- **Администратор клиники (`ADMIN`)** — управляет справочником филиалов, врачей, прайс-листом услуг и графиком смен, помогает с отменой/переносом записей.

**Внешние системы:**
- **Платёжный шлюз (ЮKassa / CloudPayments)** — принимает онлайн-оплату от клиента, возвращает ссылку на платёжную форму, отправляет вебхуки о результате оплаты и выполняет возвраты (refunds).
- **SMS / Email / Push-провайдеры (SMS.ru, SMTP, FCM)** — доставляют подтверждения записей, напоминания о визите и уведомления о готовности анализов.

### Ключевые пользовательские сценарии
1. **Как владелец питомца**, я хочу посмотреть список врачей, услуг и свободных слотов в нужном филиале, чтобы выбрать удобное время для визита.
2. **Как владелец питомца**, я хочу создать профиль своего животного (вид, порода, вес, аллергии), чтобы врач видел анамнез до начала приёма.
3. **Как владелец питомца**, я хочу записаться на приём онлайн и внести предоплату в течение 15 минут, чтобы гарантированно закрепить за собой слот врача.
4. **Как владелец питомца**, я хочу отменить запись и получить автоматический возврат предоплаты, чтобы не потерять деньги при изменении планов.
5. **Как ветеринарный врач**, я хочу после приёма внести диагноз, назначения и прикрепить PDF-анализы/рентген к медкарте питомца, чтобы история лечения сохранялась централизованно.
6. **Как владелец питомца**, я хочу автоматически получать SMS/Push о подтверждении записи, отмене и готовности результатов анализов, чтобы ничего не пропустить.
7. **Как администратор клиники**, я хочу обновлять расписание врачей и стоимость процедур в прайс-листе, чтобы клиенты всегда видели актуальные цены и слоты.

### Доменные события (упрощённый Event Storming)
- `УслугаОбновлена` / `ГрафикВрачаОбновлён`
- `ПитомецЗарегистрирован`
- `СлотУдержан` → `ЗаписьСоздана`
- `ПлатёжСоздан` → `ПлатёжПрошёл` (`payment.completed`) / `ПлатёжОтклонён` (`payment.failed`)
- `ЗаписьПодтверждена` (`appointment.confirmed`) / `ЗаписьОтменена` (`appointment.cancelled`) / `ВремяОплатыИстекло` (`appointment.expired`)
- `ВозвратЗапрошен` (`billing.refund.requested`) → `ВозвратВыполнен` (`payment.refunded`)
- `МедЗаписьДобавлена` (`medical.record.created`)
- `УведомлениеОтправлено`

### Ограниченные контексты и глоссарии

1. **Контекст «Каталог и расписание клиники» (`clinic-catalog-service`)**
   * **Услуга (`ServiceItem`)** — позиция прайс-листа с фиксированной длительностью, ценой и признаком обязательной предоплаты.
   * **Ветеринар (`Veterinarian`)** — специалист клиники со специализацией, привязкой к филиалу и графиком смен.
   * **Рабочий слот (`WorkSlot`)** — базовый временной интервал работы врача (без учёта статуса конкретной клиентской брони).
2. **Контекст «Пациенты и медкарты» (`patient-service`)**
   * **Пациент / Питомец (`Pet`)** — животное с клиническим профилем (вид, порода, вес, номер чипа, аллергии).
   * **Владелец (`Owner`)** — контактное лицо, несущее ответственность за питомца.
   * **Медицинская запись (`MedicalRecord`)** — заключение врача, диагноз, назначения и ссылки на диагностические файлы в S3.
3. **Контекст «Записи на приём» (`appointment-service`)**
   * **Запись (`Appointment`)** — забронированный визит питомца к врачу в конкретный слот со статусом жизненного цикла (`PENDING_PAYMENT`, `CONFIRMED`, `COMPLETED`, `CANCELLED`, `EXPIRED`).
   * **Удержание слота (`SlotHold`)** — временная 15-минутная блокировка слота в Redis на время оплаты.
4. **Контекст «Биллинг и платежи» (`billing-service`)**
   * **Счёт (`Invoice`)** — финансовое требование по конкретной записи на приём.
   * **Платёж (`Payment`)** — транзакция во внешнем платёжном шлюзе со статусом и ключом идемпотентности.
   * **Возврат (`Refund`)** — компенсирующая операция возврата денежных средств клиенту.
5. **Контекст «Коммуникации» (`notification-service`)**
   * **Уведомление (`Notification`)** — факт доставки сообщения клиенту по каналу (SMS, Email, Push) на основе шаблона.
   * **Шаблон (`NotificationTemplate`)** — параметризованный текст сообщения для конкретного типа события.

---

## Список и описание микросервисов

### 1. `clinic-catalog-service`
| Поле | Значение |
| :--- | :--- |
| **Название** | `clinic-catalog-service` |
| **Ответственность** | Хранение и выдача справочника филиалов, ветеринаров, прайс-листа услуг и базового расписания смен врачей. |
| **Владеет данными** | `Branch`, `Veterinarian`, `ServiceItem`, `WorkSchedule` (`catalog_db` в PostgreSQL). |
| **Предоставляет** | REST: `GET /api/v1/catalog/services`, `GET /api/v1/catalog/veterinarians`, `GET /api/v1/catalog/veterinarians/{vetId}/schedule`.<br>gRPC (`clinic_catalog.proto`): `GetServiceAndVetSlot`, `GetVeterinarian`. |
| **Зависит от** | Не зависит от других бизнес-сервисов (полностью автономен). |
| **Нефункциональные требования** | Соотношение чтений к записям ~100:1; целевая задержка p95 < 50 мс (за счёт кэширования в Redis); высокая доступность. |
| **Обоснование границы** | Разные нефункциональные требования (интенсивное чтение и кэширование отдельно от транзакционной записи); отдельная бизнес-возможность управления справочником. |

### 2. `patient-service`
| Поле | Значение |
| :--- | :--- |
| **Название** | `patient-service` |
| **Ответственность** | Ведение профилей владельцев и карточек питомцев, электронных медицинских карт, журнала прививок и диагностических вложений. |
| **Владеет данными** | `Owner`, `Pet`, `MedicalRecord`, `Vaccination`, `DiagnosticAttachment` (`patient_db` в PostgreSQL + файлы в MinIO). |
| **Предоставляет** | REST: `GET/POST /api/v1/pets`, `GET/POST /api/v1/pets/{petId}/medical-records`.<br>gRPC (`patient.proto`): `VerifyPetOwnership`.<br>Kafka (pub): `medical.record.created` (`vet.medical`). |
| **Зависит от** | gRPC → `clinic-catalog-service` (`GetVeterinarian` для отображения ФИО врача в выписке); Kafka (sub): `appointment.confirmed` (`vet.appointments`) для создания черновика визита. |
| **Нефункциональные требования** | Средняя нагрузка; строгие требования к сохранности и конфиденциальности медицинских данных; поддержка загрузки файлов до 50 МБ (снимки/PDF). |
| **Обоснование границы** | Независимая бизнес-возможность (клинический учёт и медкарта); владение медицинскими данными и файлами в S3, которые не нужны на этапе биллинга или поиска слотов. |

### 3. `appointment-service`
| Поле | Значение |
| :--- | :--- |
| **Название** | `appointment-service` |
| **Ответственность** | Онлайн-запись на приём, защита от двойного бронирования слота (удержание в Redis на 15 мин), управление жизненным циклом визита и оркестрация Саги. |
| **Владеет данными** | `Appointment`, `SlotHold`, `ProcessedEvent` (`appointment_db` в PostgreSQL + TTL-ключи в Redis). |
| **Предоставляет** | REST: `POST/GET /api/v1/appointments`, `GET /api/v1/appointments/{id}`, `POST /api/v1/appointments/{id}/cancel`.<br>Kafka (pub): `appointment.confirmed`, `appointment.cancelled`, `appointment.expired` (`vet.appointments`); команда `billing.refund.requested` (`vet.billing.commands`). |
| **Зависит от** | gRPC → `patient-service` (`VerifyPetOwnership`), `clinic-catalog-service` (`GetServiceAndVetSlot`), `billing-service` (`CreatePrepayment`); Kafka (sub): `payment.completed`, `payment.failed` (`vet.payments`). |
| **Нефункциональные требования** | Высокая критичность (ядро бизнеса); строгая защита от race condition при одновременной записи двух клиентов на один слот; p95 < 300 мс. |
| **Обоснование границы** | Ядро бизнес-логики со сложной конкурентной логикой бронирования слотов и управлением состоянием распределённой транзакции (Саги). |

### 4. `billing-service`
| Поле | Значение |
| :--- | :--- |
| **Название** | `billing-service` |
| **Ответственность** | Выставление счетов, создание онлайн-предоплат через внешний шлюз, приём и проверка криптографической подписи вебхуков, выполнение возвратов. |
| **Владеет данными** | `Invoice`, `Payment`, `Refund` (`billing_db` в PostgreSQL). |
| **Предоставляет** | REST: `GET /api/v1/invoices`, `GET /api/v1/invoices/{id}`, `POST /api/v1/payments/webhook`.<br>gRPC (`billing.proto`): `CreatePrepayment`.<br>Kafka (pub): `payment.completed`, `payment.failed`, `payment.refunded` (`vet.payments`). |
| **Зависит от** | Внешний платёжный шлюз (HTTPS); Kafka (sub): команда `billing.refund.requested` (`vet.billing.commands`). |
| **Нефункциональные требования** | Максимальная надёжность и строгие ACID-транзакции; обязательная идемпотентность всех платёжных операций; изолированный финансовый аудит. |
| **Обоснование границы** | Интеграция с внешним платёжным шлюзом, повышенные требования к безопасности и аудиту; сбой или смена шлюза не должны затрагивать логику расписания. |

### 5. `notification-service`
| Поле | Значение |
| :--- | :--- |
| **Название** | `notification-service` |
| **Ответственность** | Асинхронная отправка SMS, Email и Push-уведомлений владельцам питомцев о статусе записей, возвратах и готовности анализов. |
| **Владеет данными** | `NotificationLog`, `NotificationTemplate` (`notification_db` в PostgreSQL с полями `JSONB`). |
| **Предоставляет** | REST: `GET /api/v1/notifications`, `POST /internal/v1/notifications/send`. |
| **Зависит от** | Kafka (sub): `vet.appointments`, `vet.medical`, `vet.payments`; внешние SMS/SMTP/Push-провайдеры. |
| **Нефункциональные требования** | Eventual consistency; отказоустойчивые ретраи с Dead Letter Queue (DLQ); недоступность провайдера SMS не должна влиять на оформление записи. |
| **Обоснование границы** | Вспомогательная бизнес-возможность; слабая связность — сбой отправки SMS не должен ломать запись на приём или оплату. |

### Отвергнутые альтернативные варианты разбиения
1. **Объединение `appointment-service` и `billing-service` в один сервис** — было решено отказаться, так как финансовый модуль интегрируется с внешним платёжным шлюзом, требует отдельного аудита, строгой изоляции секретов подписи вебхуков и независимого цикла релизов при смене платёжного провайдера.
2. **Выделение `medical-records-service` отдельно от `patient-service`** — было решено отказаться, чтобы избежать излишнего дробления: медицинские записи тесно связаны с карточкой питомца (аллергии, вес, прививки), почти всегда читаются и изменяются совместно, поэтому их разделение создало бы лишние синхронные вызовы (распределённый монолит).
3. **Собственный `user-auth-service`** — было решено отказаться в пользу готового **Keycloak**, так как самостоятельная реализация OAuth 2.0 / OIDC и хранения хэшей паролей небезопасна и избыточна.

---

## Выбор инфраструктурного ПО

| Компонент ПО | Какие сервисы используют | Обоснование выбора |
| :--- | :--- | :--- |
| **PostgreSQL** (5 отдельных БД: `catalog_db`, `patient_db`, `appointment_db`, `billing_db`, `notification_db`) | Все 5 бизнес-сервисов (строго своя БД на сервис) | Требуются строгие ACID-транзакции (финансовый учёт в `billing`, уникальность слотов в `appointment`), ссылочная целостность (`patient` ↔ `medical_records`) и поддержка `JSONB` для гибких параметров шаблонов и логов в `notification-service`. Общая БД между сервисами запрещена. |
| **Redis** | `clinic-catalog-service`, `appointment-service` | В `clinic-catalog-service` снимает нагрузку на чтение прайс-листа и графиков врачей (кэш с TTL). В `appointment-service` обеспечивает атомарное удержание слота на 15 минут через `SET key value NX EX 900`, предотвращая гонку при одновременной записи. |
| **MinIO (S3-совместимое хранилище)** | `patient-service` | Хранение бинарных файлов (рентген-снимки, УЗИ, PDF-результаты анализов) вне реляционной БД PostgreSQL, чтобы не раздувать бэкапы и индексы `patient_db`. |
| **Apache Kafka** | `appointment-service`, `billing-service`, `patient-service`, `notification-service` | Надёжный брокер событий с партиционированием по `appointmentId` (гарантия порядка событий одной записи), возможностью повторного чтения лога и доставкой одного события (`appointment.confirmed`) сразу нескольким независимым подписчикам (`notification-service` и `patient-service`). |
| **gRPC (HTTP/2 + Protobuf)** | `appointment-service`, `clinic-catalog-service`, `patient-service`, `billing-service` | Высокопроизводительный бинарный протокол для внутренних синхронных вызовов с жёсткой типизацией контрактов (`.proto`), встроенными таймаутами (deadlines) и поддержкой генерации клиентов/серверов на любом языке. |
| **Kong API Gateway** | Входная точка для всех клиентов | Единая точка входа, маршрутизация REST-запросов `/api/v1/*`, валидация JWT-токенов по ключам JWKS от Keycloak, rate limiting и централизованное логирование. |
| **Keycloak** | API Gateway (и клиенты для логина) | Готовое решение OAuth 2.0 / OpenID Connect для аутентификации и ролевой модели (`OWNER`, `VET`, `ADMIN`) без написания собственного сервиса пользователей. |

---

## Схема взаимодействий и сценарии Саги

### Архитектурная схема (Container View)
<img width="3441" height="2181" alt="архитектура" src="https://github.com/user-attachments/assets/9027c612-8376-4e48-af1e-98909b473af6" />


### Таблица взаимодействий между сервисами
| № | Инициатор → Получатель | Тип | Протокол / Эндпоинт или событие | Назначение |
| :-: | :--- | :---: | :--- | :--- |
| 1 | Клиент → `clinic-catalog-service` (через Gateway) | sync | REST `GET /api/v1/catalog/services`, `/veterinarians` | Просмотр врачей, услуг и рабочих слотов |
| 2 | Клиент → `appointment-service` (через Gateway) | sync | REST `POST /api/v1/appointments` | Инициировать запись на приём |
| 3 | `appointment-service` → `patient-service` | sync | **gRPC** `PatientService.VerifyPetOwnership` | Проверить, что питомец существует и принадлежит владельцу |
| 4 | `appointment-service` → `clinic-catalog-service` | sync | **gRPC** `ClinicCatalogService.GetServiceAndVetSlot` | Проверить график врача, получить цену и длительность услуги |
| 5 | `appointment-service` → `billing-service` | sync | **gRPC** `BillingService.CreatePrepayment` | Создать счёт и получить `paymentUrl` на предоплату |
| 6 | `patient-service` → `clinic-catalog-service` | sync | **gRPC** `ClinicCatalogService.GetVeterinarian` | Получить ФИО и специализацию врача для выписки |
| 7 | `billing-service` → `appointment-service` | async | Kafka `vet.payments`: `payment.completed`, `payment.failed` | Подтвердить или отменить запись по итогам оплаты |
| 8 | `appointment-service` → `notification-service`, `patient-service` | async | Kafka `vet.appointments`: `appointment.confirmed` | Отправить подтверждение клиенту; создать черновик визита |
| 9 | `appointment-service` → `notification-service` | async | Kafka `vet.appointments`: `appointment.cancelled`, `appointment.expired` | Уведомить клиента об отмене или истечении брони |
| 10 | `appointment-service` → `billing-service` | async | Kafka `vet.billing.commands`: `billing.refund.requested` | Компенсирующая команда: вернуть деньги клиенту |
| 11 | `patient-service` → `notification-service` | async | Kafka `vet.medical`: `medical.record.created` | Уведомить владельца о готовности выписки/анализов |

### Диаграмма последовательности: успешная запись с предоплатой
<img width="3079" height="2012" alt="схема-успеха" src="https://github.com/user-attachments/assets/ed8e0a6c-71eb-46ec-a53a-bd9bcf5dff97" />

### Диаграмма последовательности: сценарии сбоя и компенсации (Сага)
<img width="3050" height="1917" alt="схема-сбоя" src="https://github.com/user-attachments/assets/d44e3811-4982-455b-bfa8-0d60bbdb89d4" />


**Описание обработки сбоев в Саге:**
1. **Оплата отклонена (`payment.failed`)**: `billing-service` публикует `payment.failed` → `appointment-service` переводит запись в `CANCELLED`, удаляет блокировку слота в Redis (слот мгновенно возвращается в продажу) и публикует `appointment.cancelled` для `notification-service`.
2. **Клиент не оплатил за 15 минут**: ключ в Redis истекает автоматически по TTL; фоновая задача в `appointment-service` переводит запись в `EXPIRED` и публикует `appointment.expired`.
3. **Поздняя оплата (после `EXPIRED`)**: `appointment-service` получает `payment.completed`, видит статус `EXPIRED` (слот уже мог занять другой клиент), **не** подтверждает запись, а отправляет компенсирующую команду `billing.refund.requested` в топик `vet.billing.commands`. `billing-service` выполняет автоматический возврат через платёжный шлюз и публикует `payment.refunded`.
4. **Сбой/таймаут gRPC при вызове `billing-service`**: `appointment-service` немедленно переводит запись в `CANCELLED`, удаляет ключ в Redis и возвращает клиенту `503 Service Unavailable`.
5. **Повторная доставка события из Kafka (дубликат)**: все потребители хранят обработанные `eventId` в локальной таблице `processed_events` в рамках той же транзакции PostgreSQL и игнорируют повторы (гарантия идемпотентности).

---

## Перечень асинхронных событий (Apache Kafka)


| Событие / Команда | Топик Kafka | Ключ партиции | Издатель | Подписчики | Схема |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `payment.completed` | `vet.payments` | `appointmentId` | `billing-service` | `appointment-service` | [`asyncapi.yaml`](api/asyncapi.yaml) |
| `payment.failed` | `vet.payments` | `appointmentId` | `billing-service` | `appointment-service` | [`asyncapi.yaml`](api/asyncapi.yaml) |
| `payment.refunded` | `vet.payments` | `appointmentId` | `billing-service` | `notification-service` | [`asyncapi.yaml`](api/asyncapi.yaml) |
| `appointment.confirmed` | `vet.appointments` | `appointmentId` | `appointment-service` | `notification-service`, `patient-service` | [`asyncapi.yaml`](api/asyncapi.yaml) |
| `appointment.cancelled` | `vet.appointments` | `appointmentId` | `appointment-service` | `notification-service` | [`asyncapi.yaml`](api/asyncapi.yaml) |
| `appointment.expired` | `vet.appointments` | `appointmentId` | `appointment-service` | `notification-service` | [`asyncapi.yaml`](api/asyncapi.yaml) |
| `billing.refund.requested` | `vet.billing.commands` | `appointmentId` | `appointment-service` | `billing-service` | [`asyncapi.yaml`](api/asyncapi.yaml) |
| `medical.record.created` | `vet.medical` | `petId` | `patient-service` | `notification-service` | [`asyncapi.yaml`](api/asyncapi.yaml) |

---

##  Контракты API

### Спецификации OpenAPI 3.1.0 (REST / Gateway + тег `internal`)
* [`api/clinic-catalog-service.yaml`](api/clinic-catalog-service.yaml) — каталог услуг, врачей и расписаний
* [`api/patient-service.yaml`](api/patient-service.yaml) — питомцы и электронные медицинские карты
* [`api/appointment-service.yaml`](api/appointment-service.yaml) — онлайн-запись, удержание слотов и отмена
* [`api/billing-service.yaml`](api/billing-service.yaml) — счета, предоплаты и вебхуки шлюза
* [`api/notification-service.yaml`](api/notification-service.yaml) — история и отправка уведомлений

### Спецификация асинхронных событий (AsyncAPI 3.0.0)
* [`api/asyncapi.yaml`](api/asyncapi.yaml) — топики Kafka, издатели, подписчики и JSON-схемы конвертов событий

### Межсервисные контракты gRPC (Protocol Buffers v3)
* [`api/proto/clinic_catalog.proto`](api/proto/clinic_catalog.proto) — `GetServiceAndVetSlot`, `GetVeterinarian`
* [`api/proto/patient.proto`](api/proto/patient.proto) — `VerifyPetOwnership`
* [`api/proto/billing.proto`](api/proto/billing.proto) — `CreatePrepayment`
