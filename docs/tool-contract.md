# Контракт Tool

Интерфейс (Python):

    class Tool(Protocol):
        name: str
        def validate(self, recipient: str, text: str) -> None: ...
        def send(self, recipient: str, text: str) -> ToolResult: ...

## Что получает
recipient (адрес в формате канала), text (непустая строка).

## Что возвращает
validate ничего не возвращает, при ошибке бросает ValidationError.
send возвращает ToolResult: ok (bool), provider_message_id (строка или null),
reason (строка или null).

## Какими ошибками отказывает
- ValidationError: адрес или текст не подходят каналу (содержит имя поля)
- TimeoutError: канал не ответил за время timeout
- UnavailableError: канал недоступен

## Кто проверяет параметры
Формат адреса и длину текста знает только Tool, поэтому проверку выполняет Tool
в методе validate. Dispatcher вызывает validate **до создания Job**. Если
validate отказал, Dispatcher возвращает клиенту 400 validation_error с полем
field, Job не создаётся. Общее (канал существует, поля не пустые) Dispatcher
проверяет сам. Метод send повторно параметры не проверяет.

## Tool 1: EmailTool
recipient это email, text до 10000 символов. validate: ValidationError при
неверном адресе. send: TimeoutError при неответе SMTP-сервера.

## Tool 2: TelegramTool
recipient это chat_id, text до 4096 символов. validate: ValidationError при
длинном тексте. send: UnavailableError, если бот заблокирован пользователем.
