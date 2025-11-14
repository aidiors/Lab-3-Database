# Индексы

---

### 1. SQL-код созданных индексов

```sql
-- Индексы для ускорения запросов к deals
CREATE INDEX idx_deals_amount_desc ON deals(amount DESC);
CREATE INDEX idx_deals_completed_amount ON deals(is_completed, amount);
CREATE INDEX idx_deals_fund_asset_crisis ON deals(fund_id, asset_id, crisis_id);

-- Индексы для economic_crises
CREATE INDEX idx_economic_crises_severity ON economic_crises(severity_score);
CREATE INDEX idx_economic_crises_start_year ON economic_crises(start_year);

-- Индексы для strategic_assets
CREATE INDEX idx_strategic_assets_value_diff ON strategic_assets(value_2008, value_2007);
CREATE INDEX idx_strategic_assets_systemic ON strategic_assets(asset_id) WHERE is_systemically_important = TRUE;
CREATE INDEX idx_strategic_assets_country_type ON strategic_assets(country_id, type_id);

-- Индексы для investment_funds
CREATE INDEX idx_funds_political_influence ON investment_funds(political_influence_score);

-- Индексы для deal_conditions
CREATE INDEX idx_deal_conditions_deal_id ON deal_conditions(deal_id);

-- Индекс для economic_indicators (оптимизация материализованного представления)
CREATE INDEX idx_indicators_country_year ON economic_indicators(country_id, year);
```

---

### 2. Сопоставление индексов с SELECT-запросами

| Запрос | Индексы | Обоснование пользы |
|--------|---------|-------------------|
| **1. Анализ крупных сделок** | `idx_deals_amount_desc`, `idx_economic_crises_severity`, `idx_deals_fund_asset_crisis` | Ускоряет фильтрацию по сумме (>1млрд) и severity_score (>=7), оптимизирует JOIN по внешним ключам, поддерживает сортировку DESC для LIMIT 5 |
| **2. Средняя стоимость по странам** | `idx_deals_completed_amount`, `idx_strategic_assets_country_type` | Ускоряет фильтрацию по is_completed=TRUE, агрегацию по amount, группировку по странам и типам активов за счет составного индекса |
| **3. Сравнение стоимости активов** | `idx_strategic_assets_value_diff`, `idx_deals_asset_id` | Оптимизирует условия WHERE (value_2008 < value_2007 и amount < value_2007*0.6), ускоряет JOIN по asset_id |
| **4. Страны с кризисами после 2007** | `idx_economic_crises_start_year`, `idx_strategic_assets_country_id` | Ускоряет фильтрацию по start_year >= 2007 в CTE, оптимизирует подсчет сделок по странам |
| **5. Системно важные активы** | `idx_strategic_assets_systemic`, `idx_funds_political_influence`, `idx_deal_conditions_deal_id` | Частичный индекс ускоряет фильтрацию по is_systemically_important=TRUE, индекс по political_influence_score >=7 ускоряет JOIN, индекс по deal_id оптимизирует агрегацию условий |

---

### 3. Анализ влияния индексов на операции записи

**Отрицательное влияние:**
- **INSERT в `deals`** (новые сделки):  
  Замедление из-за обновления 3 индексов (`amount_desc`, `completed_amount`, `fund_asset_crisis`). Особенно критично при массовой загрузке данных. Тестовая вставка для Кипра (4-й модифицирующий запрос) будет медленнее.

- **UPDATE `strategic_assets`** (второй модифицирующий запрос):  
  Обновление `value_2008` затронет индекс `value_diff`, что увеличит время выполнения для банковских активов США. Частичный индекс `systemic` не затрагивается, так как условие WHERE фильтрует только несистемные активы.

- **DELETE из `data_sources`**:  
  Не затрагивает созданные индексы, так как операция фильтрует по полям без индексов (`reliability_score`, `url`).

**Позитивные аспекты:**
- Все индексы покрывают ключевые аналитические сценарии.
- Частичные индексы (например, `systemic`) минимизируют overhead для записей, не соответствующих условию.
- Составные индексы оптимизированы под конкретные запросы, а не просто по отдельным полям.

---

### 4. Ответы на вопросы

**1. Почему не стоит создавать индекс по каждому столбцу?**  
Каждый индекс:
- Увеличивает время выполнения операций записи (INSERT/UPDATE/DELETE) из-за необходимости обновления индексных структур
- Занимает дополнительное место на диске
- Усложняет работу оптимизатора запросов, который может выбрать неоптимальный план выполнения

**2. В каких случаях индекс может ухудшить запрос?**  
Индекс вреден когда:
- Селективность поля низкая
- Запрос возвращает много строк таблицы (полное сканирование эффективнее)
- Индекс не покрывает все условия в WHERE или JOIN (требуются дополнительные обращения к таблице)
- Выполняются частые массовые обновления данных

**3. Что такое селективность столбца?**  
Селективность — отношение количества уникальных значений к общему числу записей. Высокая селективность (близкая к 1.0) означает, что индекс эффективно сокращает число обрабатываемых строк. Например:
- `iso_code` в countries: селективность = 1.0 (уникальные значения)
- `region` в countries: селективность ~0.3 (много повторов)
  Чем выше селективность, тем полезнее индекс для фильтрации данных.

**4. Что такое кардинальность столбца?**  
Кардинальность — абсолютное количество уникальных значений в столбце. В отличие от селективности, это абсолютная величина.

  Кардинальность критична при решении о создании составных индексов — поля с высокой кардинальностью должны быть первыми в составном индексе, чтобы максимизировать эффективность разделения данных.
