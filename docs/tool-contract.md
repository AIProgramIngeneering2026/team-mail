# Контракт Tool

Интерфейс (Python):
    class Tool(Protocol):
        name: str
        def send(self, recipient: str, text: str) -> ToolResult: ...

## Что получает
recipient (адрес в формате канала), text (непустая строка).

## Что возвращает
ToolResult: ok (bool), provider_message_id (строка или null), reason (строка или null).

## Какими ошибками отказывает
- ValidationError: адрес или текст не подходят каналу
- TimeoutError: канал не ответил за время timeout
- UnavailableError: канал недоступен

## Кто проверяет параметры
Tool проверяет параметры своего канала (формат адреса, длину текста).
Dispatcher проверяет только общее: канал существует, поля не пустые.

## Tool 1: EmailTool
recipient это email, text до 10000 символов. Ошибки: ValidationError при неверном
адресе, TimeoutError при неответе SMTP-сервера.

## Tool 2: TelegramTool
recipient это chat_id, text до 4096 символов. Ошибки: ValidationError при длинном
тексте, UnavailableError, если бот заблокирован пользователем.
