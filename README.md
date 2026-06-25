# Архитектура Системы Автоматической Генерации Ответов

Проект представляет собой событийно-ориентированный (event-driven) ML-пайплайн для обработки входящих обращений граждан, семантического поиска нормативно-правовых актов (RAG) и генерации официальных ответов.

---

## 🏗 Общая схема конвейера обработки

```mermaid
graph LR
    %% АКТОРЫ
    Applicant((Заявитель))
    Reviewer((Делопроизводитель))

    %% ВНЕШНИЕ СИСТЕМЫ
    Email([Почтовый сервер])
    Web([Веб-портал])
    Out([Система отправки / СЭД])

    %% БАЗЫ ДАННЫХ
    DB_Meta[(Реляционная БД)]
    DB_Vector[(Векторная БД)]
    
    %% ШИНА ДАННЫХ
    Broker{{RabbitMQ / Message Broker}}

    %% МОДУЛИ (КОНТЕЙНЕРЫ)
    M1[1. Gateway, Препроцессинг и OCR]
    M2[2. Модуль NLP]
    M3[3. Модуль RAG]
    M4[4. Модуль Генерации]
    M5[5. Постпроцессинг]

    %% ВЗАИМОДЕЙСТВИЕ НА ВХОДЕ (Синхронно)
    Applicant -->|Письмо| Email
    Applicant -->|Форма| Web
    Email -.->|Запрос| M1
    Web -.->|Запрос| M1

    %% АСИНХРОННОЕ ВЗАИМОДЕЙСТВИЕ ЧЕРЕЗ БРОКЕР
    M1 ==>|Публикация задачи| Broker
    Broker ==>|Чтение: Очищенный текст| M2
    
    M2 ==>|Публикация задачи| Broker
    Broker ==>|Чтение: Текст + Тематика| M3
    
    M3 ==>|Публикация задачи| Broker
    Broker ==>|Чтение: Текст + Чанки НПА| M4
    
    M4 ==>|Публикация задачи| Broker
    Broker ==>|Чтение: Черновик ответа| M5

    %% РАБОТА С БД И ВЫХОД
    M1 -.->|Валидация Pydantic| M1
    M2 <.->|Сохранение/Чтение ФИО| DB_Meta
    M5 <.->|Запрос реквизитов| DB_Meta
    M3 <.->|Поиск эмбеддингов| DB_Vector
    
    M5 -.->|Черновик на проверку| Reviewer
    Reviewer -.->|Утвержденный документ| Out
    Out -.->|Ответ| Applicant
    
    classDef default fill:#2b78e4,stroke:#1a52a3,stroke-width:1px,color:#fff;
    classDef db fill:#005b9f,stroke:#003d6b,stroke-width:1px,color:#fff;
    classDef ext fill:#11427b,stroke:#0a294d,stroke-width:1px,color:#fff;
    classDef broker fill:#e67e22,stroke:#d35400,stroke-width:2px,color:#fff;
    classDef actor fill:#2c3e50,stroke:#1a252f,stroke-width:1px,color:#fff;
    
    class DB_Meta,DB_Vector db;
    class Email,Web,Out ext;
    class Broker broker;
    class Applicant,Reviewer actor;
