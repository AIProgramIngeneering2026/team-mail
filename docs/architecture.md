# Архитектура

## 1. Контекст системы

```mermaid
flowchart LR
    Client[Клиентское приложение] -->|"HTTP: создать, отменить, узнать статус"| System[Сервис уведомлений]
    Admin[Администратор] -->|"HTTP: список и статусы Job"| System
    System -->|"SMTP: отправить письмо"| Mail[Почтовый сервер]
    System -->|"HTTPS: отправить сообщение"| TG[Telegram Bot API]
```

Что показывают стрелки: **вызовы** во время работы, то есть кто к кому обращается
и по какому протоколу. Стрелка идёт от вызывающего к вызываемому.

## 2. Компоненты

```mermaid
flowchart LR
    API[Входной слой HTTP] -->|"вызов: создать, отменить, получить"| D[Dispatcher]
    D -->|"вызов: сохранить и прочитать Job"| S[(Хранилище Job)]
    D -->|"вызов: найти Tool по каналу"| R[Registry]
    R -->|"выдаёт Tool email"| T1[EmailTool]
    R -->|"выдаёт Tool telegram"| T2[TelegramTool]
    T1 -->|"вызов: отправить"| A1[Адаптер почты]
    T2 -->|"вызов: отправить"| A2[Адаптер Telegram]
```

Что показывают стрелки: **вызовы** (кто кого вызывает при обработке запроса), а не
зависимости кода. По коду Dispatcher зависит только от интерфейса Tool и Registry,
конкретные Tool и адаптеры он не знает (лекция 103).

| Компонент | За что отвечает | Что скрывает от соседей |
|---|---|---|
| Входной слой HTTP | принимает запрос, проверяет формат, формирует ответ и код ошибки | устройство Dispatcher и хранилища |
| Dispatcher | создаёт Job, ведёт жизненный цикл, повторы, отмена | как именно доставляется сообщение |
| Registry | по имени канала находит нужный Tool | список и устройство Tool |
| EmailTool | проверяет адрес и текст, отправляет письмо | протокол SMTP и формат писем |
| TelegramTool | проверяет chat_id и текст, отправляет в Telegram | формат API Telegram |
| Хранилище Job | сохраняет Job и статусы | СУБД и схему таблиц |
| Адаптер почты и Telegram | вызывает внешний сервис с timeout и retry, проверяет схему ответа | библиотеку и формат API сервиса |

## 3. Диаграмма состояний Job

```mermaid
stateDiagram-v2
    [*] --> accepted: создан запрос
    accepted --> sending: начало отправки
    accepted --> cancelled: cancel
    sending --> delivered: канал подтвердил
    sending --> failed: попытки закончились
    sending --> cancelled: cancel
    delivered --> [*]
    failed --> [*]
    cancelled --> [*]
```

Состояния: accepted (принято), sending (отправляется), delivered (доставлено),
failed (не доставлено), cancelled (отменено). Из delivered, failed и cancelled
выхода нет.

### Запрещённые переходы

| Переход | Что отвечает система |
|---|---|
| delivered → sending, failed → sending, cancelled → sending | 409, код `invalid_transition`, статус не меняется |
| delivered → cancelled, failed → cancelled | 409, код `invalid_transition`, статус не меняется |
| cancelled → delivered (запоздалое подтверждение провайдера) | подтверждение игнорируется, Job остаётся cancelled |
| accepted → delivered (минуя sending) | переход невозможен, статус меняет только Dispatcher |
