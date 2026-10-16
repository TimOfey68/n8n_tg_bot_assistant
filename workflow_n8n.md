Стек: n8n (self-hosted) + PostgreSQL + pgvector + GigaChat + VPS Timeweb
Каналы: Telegram Bot API + Email (IMAP/SMTP)
Цель: обработка заявок, расчёт стоимости, формирование КП, эскалация менеджеру

Webhook (Telegram/Email)
    ↓
Извлечение идентификатора (Code)
    ↓
SQL: проверка клиента в БД
    ↓
IF: клиент авторизован?
    ├─ FALSE → Запрос авторизации → Отправка → Логирование
    └─ TRUE  → LLM: извлечение сущностей
                    ↓
               RAG: поиск регламентов в pgvector
                    ↓
               SQL/Code: расчёт стоимости
                    ↓
               LLM: формирование ответа/КП
                    ↓
               Отправка клиенту
                    ↓
               Логирование в БД
                    ↓
               IF: сумма > 500 000 ₽ или нестандарт → Эскалация менеджеру

Стек: n8n (self-hosted) + PostgreSQL + pgvector + GigaChat + VPS Timeweb
Каналы: Telegram Bot API + Email (IMAP/SMTP)
Цель: обработка заявок, расчёт стоимости, формирование КП, эскалация менеджеру

Детальное описание узлов
Нода 1. Входящий триггер
Принимает сообщение из Telegram (через setWebhook) или Email (IMAP-нода n8n).

// n8n: Telegram Trigger node
{
  "updates": ["message"],
  "additionalFields": {}
}
// На выходе: $json.message.text, $json.message.chat.id, $json.message.from.id

// n8n: Telegram Trigger node
{
  "updates": ["message"],
  "additionalFields": {}
}
// На выходе: $json.message.text, $json.message.chat.id, $json.message.from.id

Нода 2. Извлечение идентификатора клиента
Code-нода: определяет канал и формирует единый ключ для поиска в БД.

// n8n Code node (JavaScript)
const channel = $input.first().json.channel; // 'telegram' | 'email'
let identifier = null;
let chatId = null;

if (channel === 'telegram') {
  identifier = $input.first().json.message.from.id; // telegram_user_id
  chatId = $input.first().json.message.chat.id;
} else if (channel === 'email') {
  identifier = $input.first().json.from; // email отправителя
}

return [{ json: { identifier, chatId, channel,
                  text: $input.first().json.text ||
                        $input.first().json.body } }];

Нода 3. SQL: проверка клиента в БД
PostgreSQL-нода: ищет клиента по идентификатору в таблице wholesale_customers.

-- Поиск клиента по telegram_user_id или email
SELECT
  customer_id,
  company_name,
  inn,
  customer_type,     -- 'wholesale' | 'retail'
  is_verified
FROM wholesale_customers
WHERE telegram_user_id = $1
   OR email = $1
LIMIT 1;

Нода 4. IF: клиент авторизован?
Ветвление по результату SQL-запроса.

// Условие IF в n8n
{{ $json.customer_id !== undefined && $json.is_verified === true }}

// TRUE  → идём на ноду 5 (извлечение сущностей)
// FALSE → идём на ноду 4a (запрос авторизации)

Нода 4a. Запрос авторизации (если клиент не найден)

// Ответ клиенту
if (channel === 'telegram') {
  return "Здравствуйте! Для работы с оптовыми ценами, пожалуйста, укажите ваш email или ИНН.";
} else {
  return "Ваш email не найден в базе оптовых клиентов. Для регистрации обратитесь к менеджеру.";
}

Нода 5. LLM (GigaChat): извлечение сущностей
Из текста клиента извлекаем: товар, количество, город доставки. ПДн в LLM не передаём.

// Системный промпт для GigaChat
"Ты — извлекатель сущностей. Из текста запроса клиента извлеки:
- product (название или артикул товара)
- quantity (количество, число)
- city (город доставки)
Верни JSON: {"product": "...", "quantity": ..., "city": "..."}
Если поле не найдено — поставь null. Не выдумывай."

// User-промпт (только текст запроса, без ПДн)
"Нужно 500 листов ЦСП 8мм в Москву"

// Ожидаемый ответ LLM:
{ "product": "ЦСП 8мм", "quantity": 500, "city": "Москва" }

Нода 6. RAG: поиск регламентов доставки в pgvector
Эмбеддинг запроса → поиск 3–5 релевантных чанков регламентов в delivery_regulations.

-- 1. Получаем эмбеддинг запроса (через GigaChat Embeddings API)
-- 2. Поиск ближайших чанков по косинусному расстоянию
SELECT
  chunk_id,
  content,
  metadata->>'region' AS region,
  1 - (embedding <=> $1::vector) AS similarity
FROM delivery_regulations
WHERE metadata->>'region' IN ($2, 'РФ')  -- регион клиента + общие правила
ORDER BY embedding <=> $1::vector
LIMIT 5;

Нода 7. SQL/Code: расчёт стоимости
LLM не считает! Математика — только в SQL/Code. Розница/опт определяется по customer_type.

-- Получение цены по артикулу и типу клиента
SELECT
  p.article,
  p.name,
  CASE
    WHEN $1 = 'wholesale' THEN p.price_wholesale
    ELSE p.price_retail
  END AS unit_price
FROM products p
WHERE p.article = $2
   OR p.name ILIKE $3;

// n8n Code node: итоговый расчёт
const unitPrice = $json.unit_price;
const qty = $json.quantity;
const deliveryTariff = $json.delivery_tariff; // из RAG-чанка

const goodsTotal = unitPrice * qty;
const deliveryTotal = deliveryTariff;
const grandTotal = goodsTotal + deliveryTotal;

return [{ json: {
  goodsTotal: goodsTotal.toFixed(2),
  deliveryTotal: deliveryTotal.toFixed(2),
  grandTotal: grandTotal.toFixed(2),
  currency: 'RUB'
}}];

Нода 8. LLM (GigaChat): формирование ответа / КП
Передаём в LLM только готовые данные (сущности + расчёты + регламенты). ПДн — не передаём.

// Системный промпт
"Ты — ассистент отдела продаж стройматериалов.
Сформируй вежливый ответ клиенту на основе предоставленных данных.
Не выдумывай цены, наличие, условия. Не повторяй ИНН, email, телефоны."

// User-промпт (готовые данные)
"Данные для ответа:
- Товар: ЦСП 8мм
- Количество: 500 листов
- Цена за единицу: 850 ₽
- Стоимость товара: 425 000 ₽
- Доставка в Москву: 15 000 ₽
- Итого: 440 000 ₽
- Срок доставки: 3-5 дней
- Условие оплаты: 100% предоплата для новых клиентов"

Сформируй ответ для Telegram (кратко, 2-4 предложения).

Нода 9. Отправка ответа клиенту

// Telegram: HTTP Request node
POST https://api.telegram.org/bot{TOKEN}/sendMessage
{
  "chat_id": {{ $json.chatId }},
  "text": {{ $json.response_text }},
  "parse_mode": "Markdown"
}

// Email: SMTP node (для email-канала)

Нода 10. Логирование диалога в БД (с маскированием ПДн)

// n8n Code node: маскирование перед записью
function maskINN(inn) {
  if (!inn || inn.length < 4) return inn;
  return '***-**-***' + inn.slice(-2);
}
function maskPhone(p) {
  if (!p) return p;
  return p.slice(0, 4) + '***-***-' + p.slice(-4);
}
function maskEmail(e) {
  if (!e) return e;
  const [name, domain] = e.split('@');
  return name[0] + '***@' + domain;
}

return [{ json: {
  chat_id: $json.chatId,
  customer_id: $json.customer_id,
  channel: $json.channel,
  client_text: $json.text,
  bot_response: $json.response_text,
  inn_masked: maskINN($json.inn),
  phone_masked: maskPhone($json.phone),
  email_masked: maskEmail($json.email),
  created_at: new Date()
}}];

-- Запись в логи
INSERT INTO dialog_logs
  (chat_id, customer_id, channel, client_text, bot_response,
   inn_masked, phone_masked, email_masked, created_at)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9);



