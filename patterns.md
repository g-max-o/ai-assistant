# Каталог паттернов проектирования

Документ содержит обязательные для применения паттерны и рекомендации по их реализации
в Python-коде. Все примеры учитывают асинхронность и типизацию.

## 1. Repository

Абстрагирует доступ к хранилищу данных. Клиентский код работает только с доменными объектами.

```python
from abc import ABC, abstractmethod
from uuid import UUID
from domain.entities import User
from domain.value_objects import Email

class UserRepository(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: UUID) -> User | None: ...

    @abstractmethod
    async def get_by_email(self, email: Email) -> User | None: ...

    @abstractmethod
    async def add(self, user: User) -> None: ...
```
Реализация в инфраструктуре использует ORM, возвращает доменные объекты, скрывая детали SQL.

## 2. Unit of Work
Управляет транзакциями и отслеживанием изменений. Предоставляет доступ к репозиториям.

```python
from typing import Protocol
from domain.interfaces import UserRepository, OrderRepository

class UnitOfWork(Protocol):
    users: UserRepository
    orders: OrderRepository

    async def commit(self) -> None: ...
    async def rollback(self) -> None: ...
```
Используется в сервисах приложения для атомарных операций. Контекстный менеджер (async with) обеспечивает автоматический откат при исключении.

## 3. Dependency Injection
Все зависимости внедряются через конструктор или фабрику. В FastAPI активно используется Depends().

```python
from fastapi import Depends
from infrastructure.database.unit_of_work import SqlAlchemyUnitOfWork

def get_uow() -> SqlAlchemyUnitOfWork:
    return SqlAlchemyUnitOfWork(session_factory)

@app.post("/users")
async def create_user(uow: SqlAlchemyUnitOfWork = Depends(get_uow)):
    ...
```
Избегайте глобальных синглтонов – всё через DI.

## 4. Factory
Создание сложных объектов, особенно с большим количеством параметров или конфигураций.

```python
class PaymentGatewayFactory:
    @staticmethod
    def create(provider: str) -> PaymentGateway:
        if provider == "stripe":
            return StripeAdapter(config=...)
        ...
```
Используйте фабрики внутри DI, чтобы возвращать конкретную реализацию интерфейса.

## 5. Strategy
Определяет семейство алгоритмов, инкапсулирует каждый и делает их взаимозаменяемыми.

Пример: расчёт стоимости доставки в зависимости от службы (почта, курьер, самовывоз).
Каждая стратегия реализует общий интерфейс ShippingStrategy.

## 6. Decorator
Для сквозной функциональности (логирование, кэширование, метрики). Обязательно используйте functools.wraps.

```python
import functools
import time
from loguru import logger

def timing(func):
    @functools.wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = await func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        logger.debug(f"{func.__name__} executed in {elapsed:.4f}s")
        return result
    return wrapper
```

## 7. Chain of Responsibility
Применяется для обработки запросов цепочкой middleware (аутентификация, логирование, CORS).

В FastAPI – через порядок добавления middleware в приложение. Каждый обработчик может прервать цепочку или передать запрос дальше.

## 8. Observer / Event-driven
Для слабосвязанной передачи сообщений между компонентами. Используйте pyee или простые кастомные сигналы.

```python
from pyee.asyncio import AsyncIOEventEmitter

emitter = AsyncIOEventEmitter()
emitter.on('order_created', send_confirmation_email)
await emitter.emit('order_created', order=order)
```
## 9. Adapter
Адаптирует интерфейс внешнего сервиса к нужному формату. Всегда оборачивайте сторонние SDK/API в свои классы.

```python
from application.interfaces import SmsSender
from some_sms_lib import Client

class SmsLibAdapter(SmsSender):
    def __init__(self, client: Client) -> None:
        self._client = client

    async def send(self, phone: str, message: str) -> None:
        await self._client.send_sms(phone, message)
```
## 10. Facade
Упрощает работу со сложной подсистемой. Например, ReportGenerator объединяет несколько сервисов (шаблоны, экспорт, хранилище).

Все публичные методы фасада – это единственная точка входа для контроллеров.

## Принципы реализации
Каждый паттерн должен быть выражен через интерфейс (Protocol/ABC) и его конкретную реализацию.

Не применяйте паттерны ради паттернов – только если они решают реальную проблему сложности или изменяемости.

Именуйте классы согласно роли паттерна: *Repository, *Adapter, *Strategy.
