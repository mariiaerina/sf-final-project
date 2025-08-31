# Задание 1

```sql

-- Расчет rolling retention с разбивкой по когортам

WITH base AS (
    SELECT
        to_char(u.date_joined, 'YYYY-MM') AS yw,
        count(DISTINCT u.id) AS cohort_size
    FROM users u 
    GROUP BY yw
),
user_activity AS (
    SELECT
        u.id,
        to_char(u.date_joined, 'YYYY-MM') AS yw,
        extract(DAY FROM (ue.entry_at - u.date_joined)) AS days_diff
    FROM users u 
    JOIN userentry ue ON u.id = ue.user_id
    WHERE ue.entry_at >= u.date_joined
)
SELECT
    ua.yw,
    round((100.00 * count(DISTINCT CASE WHEN days_diff >= 0 THEN id ELSE NULL END) / b.cohort_size), 2) AS "day0",
    round((100.00 * count(DISTINCT CASE WHEN days_diff >= 1 THEN id ELSE NULL END) / b.cohort_size), 2) AS "day1",
    round((100.00 * count(DISTINCT CASE WHEN days_diff >= 3 THEN id ELSE NULL END) / b.cohort_size), 2) AS "day3",
    round((100.00 * count(DISTINCT CASE WHEN days_diff >= 7 THEN id ELSE NULL END) / b.cohort_size), 2) AS "day7",
    round((100.00 * count(DISTINCT CASE WHEN days_diff >= 14 THEN id ELSE NULL END) / b.cohort_size), 2) AS "day14",
    round((100.00 * count(DISTINCT CASE WHEN days_diff >= 30 THEN id ELSE NULL END) / b.cohort_size), 2) AS "day30",
    round((100.00 * count(DISTINCT CASE WHEN days_diff >= 60 THEN id ELSE NULL END) / b.cohort_size), 2) AS "day60",
    round((100.00 * count(DISTINCT CASE WHEN days_diff >= 90 THEN id ELSE NULL END) / b.cohort_size), 2) AS "day90"
FROM user_activity ua
JOIN base b ON ua.yw = b.yw 
GROUP BY ua.yw, b.cohort_size
ORDER BY ua.yw ASC ;
```

Выводы:

В результате запроса можно наблюдать достаточно низкую долю долгосрочно удерживаемых пользователей. Для большинства когорт отток составляет более 90% к 90-му дню.

# Задание 2

```sql

-- Расчет метрик относительно баланса пользователя

WITH user_balance_stats AS (
    SELECT
        user_id,
        sum(CASE WHEN type_id IN (1, 23, 24, 25, 26, 27, 28, 30) THEN value ELSE 0 END) AS total_debit,
        sum(CASE WHEN type_id NOT IN (1, 23, 24, 25, 26, 27, 28, 30) THEN value ELSE 0 END) AS total_credit,
        sum(value) AS current_balance
    FROM "transaction" t
    GROUP BY user_id
)
SELECT 
    count(DISTINCT user_id) AS total_users,
    round(avg(total_debit), 2) AS avg_coins_debited_per_user,
    round(avg(total_credit), 2) AS avg_coins_credited_per_user,
    round(avg(current_balance), 2) AS avg_user_balance,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY current_balance) AS median_user_balance
FROM user_balance_stats ubs ;
```

Выводы: 

Данные демонстрируют, что пользователи привыкли совершать достаточно дорогие покупки, списания гораздо более существенные, чем начисления, а поэтому можно предлагать достаточно высокий прайс для подписки.

# Задание 3

```sql

-- Расчет метрик активности пользователей на платформе

WITH
-- Общее количество пользователей
total_users AS (
    SELECT count(*) AS cnt FROM users
),
-- Пользователи, решавшие задачи
task_users AS (
    SELECT DISTINCT user_id FROM codesubmit
),
-- Пользователи, проходившие тесты
test_users AS (
    SELECT DISTINCT user_id FROM teststart
),
-- Активные пользователи (решали задачи или тесты)
active_users AS (
    SELECT DISTINCT user_id FROM task_users
    UNION
    SELECT DISTINCT user_id FROM test_users
),
-- Статистика по задачам
task_stats AS (
    SELECT
        user_id,
        count(DISTINCT problem_id) AS tasks_attempted,
        count(*) AS total_task_attempts
    FROM codesubmit
    GROUP BY user_id
),
-- Статистика по тестам
test_stats AS (
    SELECT
        user_id,
        count(DISTINCT test_id) AS tests_attempted,
        count(*) AS total_test_attempts
    FROM teststart
    GROUP BY user_id 
),
-- Статистика покупок за кодкоины
purchase_stats AS (
    SELECT
        count(DISTINCT CASE WHEN tt."type" = 23 THEN t.user_id END) AS users_bought_tasks,
        count(DISTINCT CASE WHEN tt."type" = 27 THEN t.user_id END) users_bought_tests,
        count(DISTINCT CASE WHEN tt."type" = 24 THEN t.user_id END) users_bought_hints,
        count(DISTINCT CASE WHEN tt."type" = 25 THEN t.user_id END) AS users_bought_solutions,
        count(CASE WHEN tt."type" = 23 THEN t.id END) AS tasks_bought,
        count(CASE WHEN tt."type" = 27 THEN t.id END) AS tests_bought,
        count(CASE WHEN tt."type" = 24 THEN t.id END) AS hints_bought,
        count(CASE WHEN tt."type" = 25 THEN t.id END) AS solutions_bought,
        count(DISTINCT CASE WHEN tt."type" IN (1, 23, 24, 25, 26, 27, 28, 30) THEN t.user_id END) AS users_purchased_anything,
        count(DISTINCT t.user_id) AS users_with_any_transaction
    FROM "transaction" t 
    JOIN transactiontype tt ON t.type_id = tt."type"
),
aggregated_stats AS (
    SELECT 
        count(DISTINCT tks.user_id) AS task_users_count,
        coalesce(avg(tks.tasks_attempted), 0) AS avg_tasks_per_user,
        coalesce(avg(tks.total_task_attempts), 0) AS avg_task_attempts_per_user,
        coalesce(avg(tks.total_task_attempts * 1.0 / NULLIF(tks.tasks_attempted, 0)), 0) AS avg_attempts_per_task,
        count(DISTINCT tts.user_id) AS test_users_count,
        coalesce(avg(tts.tests_attempted), 0) AS avg_tests_per_user,
        coalesce(avg(tts.total_test_attempts), 0) AS avg_test_attempts_per_user,
        coalesce(avg(tts.total_test_attempts * 1.0 / NULLIF(tts.tests_attempted, 0)), 0) AS avg_attempts_per_test
    FROM task_stats tks
    FULL OUTER JOIN test_stats tts ON tks.user_id = tts.user_id 
)
SELECT
    -- Метрики активности
    (SELECT count(*) FROM active_users au) AS active_users_count,
    round((SELECT count(*) FROM active_users) * 100.00 / nullif((SELECT cnt FROM total_users), 0), 2) AS active_users_percent,
    -- Средние показатели по задачам
    round(ags.avg_tasks_per_user, 2) AS avg_tasks_per_user,
    round(ags.avg_task_attempts_per_user, 2) AS avg_task_attempts_per_user,
    round(ags.avg_attempts_per_task, 2) AS avg_attempts_per_task,
    ags.task_users_count,
    -- Средние показатели по тестам
    round(ags.avg_tests_per_user, 2) AS avg_tests_per_user,
    round(ags.avg_test_attempts_per_user, 2) AS avg_test_attempts_per_user,
    round(ags.avg_attempts_per_test, 2) AS avg_attempts_per_test,
    ags.test_users_count,
    -- Статистика покупок
    ps.users_bought_tasks,
    ps.users_bought_tests,
    ps.users_bought_hints,
    ps.users_bought_solutions,
    ps.tasks_bought,
    ps.tests_bought,
    ps.hints_bought,
    ps.solutions_bought,
    ps.tasks_bought + ps.tests_bought + ps.hints_bought + ps.solutions_bought AS total_items_purchased,
    ps.users_purchased_anything,
    ps.users_with_any_transaction,
    round(ps.users_purchased_anything * 100.00 / NULLIF((SELECT cnt FROM total_users), 0), 2) AS percent_users_purchased_anything
FROM aggregated_stats ags
CROSS JOIN purchase_stats ps
CROSS JOIN total_users tu ;
```

Выводы:

Платформа демонстрирует высокую вовлеченность (62% активных пользователей) и исключительно успешную монетизацию (41% пользователей совершали покупки). Основной рост видится в стимулировании неактивных пользователей и повышении вовлеченности в тестах, которые проходят массово, но поверхностно.
Касаемо функционала подписки, имеет смысл разделить её на базовый и премиальный пакет. В базовом пакете можно предоставить неограниченный доступ к тестам и 1-2 сложным задачам, что покроет потребности большинства пользователей, которые активно покупают тесты, но мало их решают.
В премиум-подписку же можно ввести неограниченные подсказки и решения к задачам. Это позволит монетизировать потребности тех пользователей, которые точечно покупают данные опции, и увеличить средний чек покупки.