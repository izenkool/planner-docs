# Описание моделей данных

## Модель `tasks`
Представляет собой задачу в планировщике.

### Поля:

*   **`id`** (uuid, primary key)
    *   Уникальный идентификатор задачи.
*   **`title`** (string, 255)
    *   Название задачи.
*   **`description`** (text)
    *   Подробное описание задачи.
*   **`priority`** (integer)
    *   Приоритет задачи.
*   **`created_at`** (datetime)
    *   Дата и время создания задачи (устанавливается автоматически).
*   **`updated_at`** (datetime)
    *   Дата и время последнего обновления задачи (обновляется автоматически).
*   **`completed_at`** (datetime, nullable)
    *   Дата и время завершения задачи.
*   **`deadline`** (datetime, nullable)
    *   Закончить обязательно до
*   **`starts_at`** (datetime, nullable)
    *   Приступить не раньше чем
*   **`recurrence_rules_id`** (uuid, FOREIGN KEY to `recurrence_rules.id`)
    *   Правило повторения

## Модель `tags`
Представляет собой тег, который можно присвоить задаче.

### Поля:

*   **`id`** (uuid, primary key)
    *   Уникальный идентификатор тега.
*   **`name`** (string, 255, unique)
    *   Уникальное имя тега.

## Связующая таблица `task_tag`
Для реализации связи "многие-ко-многим" между `task` и `tag` создается таблица `task_tag` со следующими полями:

*   `task_id` (uuid, FOREIGN KEY to `tasks.id`)
*   `tag_id` (uuid, FOREIGN KEY to `tags.id`)
*   **Составной первичный ключ:** (`task_id`, `tag_id`)

## Модель `recurrence_rules`
Представляет собой правила повторения для задач.

### Поля:

*   **`id`** (uuid, primary key)
    *   Уникальный идентификатор правила.
*   **`recurrence_type`** (enum: 'daily', 'weekly', 'monthly')
    *   Тип повторения.
*   **`interval`** (integer)
    *   Интервал повторения (каждые N периодов).
*   **`days_of_week`** (integer, 7 бит)
    *   Битовая маска для дней недели.
*   **`days_of_month`** (integer, 31 бит)
    *   Битовая маска для дней месяца.
*   **`active`** (boolean)
    *   Указывает, активно ли правило повторения.

## Модель `tasks_days`
Привязка задач к определенным дням

### Поля:
*   `task_id` (INTEGER, FOREIGN KEY to `tasks.id`)
*   `day` (date)
*   `complete` (boolean, default false)