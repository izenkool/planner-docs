## Общая идея

- Для ежедневных задач используется тип `daily`.
    
- Для повторения по дням недели — `weekly`, с битовой маской в поле `days_of_week`.
    
- Для повторения по числам месяца — `monthly`, с битовой маской в поле `days_of_month`.
    
- Интервал задаёт через сколько периодов (дней, недель, месяцев) повторять задачу.
    
- Признак `active` определяет, активно ли правило на текущий момент.

## Алгоритм назначения задач на конкретный день

Допустим, нужно получить все задачи, которые повторяются на конкретную дату `date`.

1. Игнорируем все неактивные правила (`active = false`).
    
2. В зависимости от типа повторения:
    
    - `daily`: Подсчитываем количество дней с некоторой условной эпохи (например, 1 января 1970). Если это число делится на `interval` без остатка, задача должна повториться.
        
    - `weekly`: Подсчитываем количество полных недель с эпохи. Если число недель делится на `interval` и установлен бит, соответствующий дню недели текущей даты в `days_of_week`, задача повторяется.
        
    - `monthly`: Подсчитываем количество месяцев с эпохи. Если число месяцев делится на `interval` и установлен бит, соответствующий дню месяца в `days_of_month`, задача включается.
        

## Псевдокод определения активных задач на дату

```
function getTasksForDate(date):
    result_tasks = []

    for each rule in recurrence_rules:
        if not rule.active:
            continue

        if rule.recurrence_type == 'daily':
            days_diff = daysSinceEpoch(date)
            if days_diff % rule.interval == 0:
                result_tasks.append(rule.task_id)

        else if rule.recurrence_type == 'weekly':
            weeks_diff = weeksSinceEpoch(date)
            if weeks_diff % rule.interval == 0:
                weekday_bit = 1 << dayOfWeek(date) # 0 - понедельник
                if rule.days_of_week & weekday_bit != 0:
                    result_tasks.append(rule.task_id)

        else if rule.recurrence_type == 'monthly':
            months_diff = monthsSinceEpoch(date)
            if months_diff % rule.interval == 0:
                day_bit = 1 << (dayOfMonth(date) - 1)
                if rule.days_of_month & day_bit != 0:
                    result_tasks.append(rule.task_id)

    return fetchTasksByIds(result_tasks)
```

## Пример значений битовых масок
| Тип           | Значение                                  | Описание                 |
| ------------- | ----------------------------------------- | ------------------------ |
| days_of_week  | 0b0100001 (33)                            | Понедельник и суббота    |
| days_of_month | 0b0000000000001000000000000000000 (32768) | Только 16-е число месяца |

## Пример создания таблиц и выборки повторяющихся задач на конкретную дату в PostgreSQL
```
-- Таблица задач
CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Тип для типа повторения
CREATE TYPE recurrence_type_enum AS ENUM ('daily', 'weekly', 'monthly');

-- Таблица правил повторения
CREATE TABLE recurrence_rules (
    id SERIAL PRIMARY KEY,
    task_id INT NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    recurrence_type recurrence_type_enum NOT NULL,
    interval INT NOT NULL DEFAULT 1,         -- через сколько периодов повторять
    days_of_week INT DEFAULT 0,              -- 7 бит для дней недели
    days_of_month INT DEFAULT 0,             -- 31 бит для дней месяца
    active BOOLEAN NOT NULL DEFAULT TRUE     -- активность правила
);

-- Функции вычисления количества дней от эпохи (1970-01-01)
CREATE OR REPLACE FUNCTION days_since_epoch(d DATE) RETURNS INT AS $$
BEGIN
    RETURN (d - DATE '1970-01-01')::INT;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Количество недель от эпохи (полные недели)
CREATE OR REPLACE FUNCTION weeks_since_epoch(d DATE) RETURNS INT AS $$
BEGIN
    RETURN FLOOR(EXTRACT(EPOCH FROM (d - DATE '1970-01-01')) / (7 * 86400))::INT;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Количество месяцев от эпохи (год * 12 + месяц)
CREATE OR REPLACE FUNCTION months_since_epoch(d DATE) RETURNS INT AS $$
BEGIN
    RETURN (EXTRACT(YEAR FROM d)::INT - 1970) * 12 + (EXTRACT(MONTH FROM d)::INT - 1);
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Запрос для получения задач на указанную дату :date
WITH date_tasks AS (
    SELECT r.task_id
    FROM recurrence_rules r
    WHERE r.active = TRUE
      AND (
         (r.recurrence_type = 'daily' AND (days_since_epoch(:date) % r.interval) = 0)
         OR
         (r.recurrence_type = 'weekly' AND (weeks_since_epoch(:date) % r.interval) = 0
          AND (r.days_of_week & (1 << ((EXTRACT(DOW FROM :date)::INT + 6) % 7))) != 0)
         OR
         (r.recurrence_type = 'monthly' AND (months_since_epoch(:date) % r.interval) = 0
          AND (r.days_of_month & (1 << (EXTRACT(DAY FROM :date)::INT - 1))) != 0)
      )
)
SELECT t.*
FROM tasks t
JOIN date_tasks dt ON t.id = dt.task_id;
```

Объяснение:

- Биты дней недели (days_of_week) нумеруются с 0 для понедельника до 6 для воскресенья. PostgreSQL `EXTRACT(DOW)` возвращает 0 для воскресенья, поэтому добавляем +6 и берём по модулю 7, чтобы согласовать.
    
- Для дней месяца (days_of_month) бит 0 — это 1-е число месяца.
    
- Интервал задаёт, через сколько дней, недель или месяцев повторять задачу.
    
- Признак active включает или отключает правило повторения.

- Вместо :date подставляется интересующая дата (например, '2025-09-25').

- Такой подход эффективен по использованию памяти и быстр по вычислениям, так как битовые операции выполняются быстро.