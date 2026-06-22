# технічна специфікація для рефакторингу системи PerspectaMind під потреби вашого middleware-компонента в Antigravity 2.0 (через CLI agy) з використанням моделей deepseek-v4-pro.

Технічна специфікація: Інтеграція PerspectaMind як Middleware в Antigravity 2.0 (DeepSeek Backend)
1. Конфігурація середовища та API-клієнтів
Оригінальна архітектура PerspectaMind використовує бібліотеку litellm та модулі DSPy для маршрутизації запитів до LLM. У нових версіях PerspectaMind було додано нативну підтримку DeepSeekModel. Для роботи в середовищі Antigravity 2.0 необхідно розширити базовий файл конфігурації.  

Змінні середовища:

Code snippet
DEEPSEEK_API_KEY="your_deepseek_api_key"
DEEPSEEK_API_BASE="https://api.deepseek.com/v1"
При ініціалізації класу PerspectaMindWikiLMConfigs необхідно перевизначити всі три основні моделі (дослідження, структурування, генерація) на об'єкти DeepSeek:

Python
# Ініціалізація моделей для конвеєра Antigravity 2.0
deepseek_kwargs = {
    "api_key": os.getenv("DEEPSEEK_API_KEY"),
    "api_base": os.getenv("DEEPSEEK_API_BASE"),
    "temperature": 1.0,
    "max_tokens": 8000
}

# Використання єдиної thinking-моделі для всіх етапів
lm_configs = PerspectaMindWikiLMConfigs()
lm_configs.set_conv_simulator_lm(DeepSeekModel(model='ddeepseek-v4-pro', **deepseek_kwargs))
lm_configs.set_question_asker_lm(DeepSeekModel(model='ddeepseek-v4-pro', **deepseek_kwargs))
lm_configs.set_outline_gen_lm(DeepSeekModel(model='ddeepseek-v4-pro', **deepseek_kwargs))
lm_configs.set_article_gen_lm(DeepSeekModel(model='ddeepseek-v4-pro', **deepseek_kwargs))
lm_configs.set_article_polish_lm(DeepSeekModel(model='ddeepseek-v4-pro', **deepseek_kwargs))
2. Адаптація параметрів генерації для Thinking-моделей
Моделі з прихованим ланцюжком міркувань (Chain-of-Thought), такі як deepseek-v4-pro, вимагають специфічних налаштувань для уникнення деградації продуктивності.  

Температура: Як і для моделей серії OpenAI o1, параметр temperature повинен бути встановлений на рівні 1.0, що дозволяє моделі ефективно досліджувати альтернативні шляхи у внутрішніх міркуваннях.  

Контекст та токени виводу: Параметр max_tokens (або max_completion_tokens) має бути не менше 5000–8000. Це критично важливо, оскільки моделі DeepSeek генерують великі блоки міркувань перед видачею фінального результату.  

3. Рефакторинг обробки виводу (DSPy Парсинг та теги міркування)
Моделі deepseek-v4-pro виводять свій внутрішній процес мислення у форматі <think>...</think> перед основною відповіддю. Це гарантовано ламає стандартні регулярні вирази ChatAdapter або JSONAdapter у бібліотеці DSPy, яка лежить в основі PerspectaMind.
Для Antigravity 2.0 потрібно впровадити спеціалізований перехоплювач на рівні DSPy-адаптера:  

Створити або налаштувати тип dspy.Reasoning, який повідомляє адаптеру, що міркування слід отримувати з відповідного поля або парсити, ігноруючи блок <think> при валідації JSON чи розмітки.  

Увімкнути резервний механізм use_json_adapter_fallback=True у налаштуваннях DSPy. Якщо модель порушить формат під час генерації кінцевого виводу, адаптер автоматично повторить запит із жорсткішим примусом до JSON.  

4. Заміна базових DSPy-сигнатур PerspectaMind (Суворе якорування)
Щоб система працювала як універсальний middleware, а не генератор вікі-статей, необхідно переписати оригінальні класи спадкоємців dspy.Signature.

У файлі persona_generator.py скасувати виклик методу create_writer_with_persona, який шукає пов'язані теми та генерує ролі на кшталт "Wikipedia Writer". Замінити його на функціональний класифікатор компетенцій на основі вашої змінної main_question.  

У файлі outline_generator.py (клас WritePageOutline) вилучити вимогу до форматування заголовків символами #. Оригінальна функція clean_up_outline() схильна до збоїв парсингу. Її необхідно переписати під обробку суворого JSON-виводу логічної структури відповіді.  

5. Налаштування ретриверів (Retrievers)
Оскільки deepseek-v4-pro здатен здійснювати глибокий аналіз складних даних, якість подачі інформації через модулі knowledge_perspecta_mind/rm.py є критичною.

При роботі із зовнішніми даними підключіть YouRM або DuckDuckGoSearchRM, але врахуйте необхідність обробки помилок парсингу API (наприклад, відловлювання AssertionError: keywords is mandatory для DuckDuckGo).  

При роботі з корпоративними даними Antigravity 2.0 ініціалізуйте VectorRM для завантаження документів, гарантуючи, що кожному документу присвоєно унікальний url, інакше система відкине їх під час агрегації. Обов'язково використовуйте аргумент для оновлення векторної бази (наприклад, --update-vector-store), щоб система мала доступ до актуального контексту.


2. Порівняльний аналіз провайдерів LLM

Система підтримує два взаємозамінні канали роботи з LLM, вибір між якими регулюється змінною оточення RND_LLM_PROVIDER у файлі `.env` (значення `deepseek` або `antigravity_cli`).

### А. Інтеграція через DeepSeek API (REST HTTP)
- **Функція реалізації**: `call_deepseek`
- **Механізм роботи**: Використовує асинхронний Rust-клієнт `reqwest` для виконання HTTP POST запитів до офіційного ендпоінту `https://api.deepseek.com/v1/chat/completions`.
- **Авторизація**: Зчитує `DEEPSEEK_API_KEY` із системного оточення.
- **Мапінг моделей**: За замовчуванням використовуються моделі `deepseek-v4-pro` (для синтезу та архітектури) та `deepseek-v4-flash` (для критики та аудиту). За потреби їх можна перевизначити через змінні оточення (наприклад, `RND_SKEPTIC_MODEL_DEEPSEEK`).
- **Облік токенів**: Отримує точні показники споживання безпосередньо з JSON-відповіді сервера (`usage.prompt_tokens` + `usage.completion_tokens`) та логує їх у файл `logs/rnd_deepseek_token_usage.log`.

### Б. Інтеграція через Antigravity CLI (agy)
- **Функція реалізації**: `call_antigravity_cli`
- **Механізм роботи**: Запускає локальну CLI-утиліту `agy` через `std::process::Command` всередині віртуального терміналу `script -q -c` (для уникнення проблем з буферизацією введення-виведення).
- **Команда запуску**:
  ```bash
  /home/lden_eons_28cores/.local/bin/agy --model <model> --dangerously-skip-permissions -p '<system_prompt + prompt>'
  ```
  *(Спеціальні символи та одинарні лапки в промпті автоматично екрануються).*
- **Мапінг моделей**: Визначається через змінні `RND_<ROLE>_MODEL_ANTIGRAVITY`. За замовчуванням використовуються моделі лінійки `gemini-3.5-flash-low`, `gemini-3.5-flash-medium` та `gemini-3.1-pro-high` / `gemini-3.1-pro-low`.
- **Облік токенів**: Оскільки CLI-інтерфейс не повертає метаданих API, споживання оцінюється локально за допомогою бібліотеки `tiktoken_rs` (кодування `cl100k_base`). Для української кирилиці реалізовано резервний евристичний алгоритм: `(кількість слів * 2.5)`. Дані логуються у файл `logs/rnd_antigravity_token_usage.log`.

### В. Порівняльна таблиця провайдерів

| Характеристика | DeepSeek API (REST HTTP) | Antigravity CLI (agy) |
| :--- | :--- | :--- |
| **Спосіб доступу** | Мережевий REST API через `reqwest` | Локальний запуск CLI (`std::process::Command`) |
| **Ендпоінт / Команда** | `https://api.deepseek.com/v1/chat/completions` | `/home/lden_eons_28cores/.local/bin/agy` |
| **Авторизація** | `DEEPSEEK_API_KEY` (токен доступу) | Локальні права виконання та спеціальні прапорці CLI |
| **Моделі за замовчуванням** | `deepseek-v4-pro`, `deepseek-v4-flash` | `gemini-3.5-flash-low/medium`, `gemini-3.1-pro-high/low` |
| **Облік токенів** | Точний (з JSON-відповіді сервера) | Оціночний (`tiktoken_rs` + евристика `слова * 2.5` для кирилиці) |
| **Логування токенів** | `logs/rnd_deepseek_token_usage.log` | `logs/rnd_antigravity_token_usage.log` |
