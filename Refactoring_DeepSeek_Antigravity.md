# технічна специфікація для рефакторингу системи STORM під потреби вашого middleware-компонента в Antigravity 2.0 (через CLI agy) з використанням моделей deepseek-v4-pro.

Технічна специфікація: Інтеграція STORM як Middleware в Antigravity 2.0 (DeepSeek Backend)
1. Конфігурація середовища та API-клієнтів
Оригінальна архітектура STORM використовує бібліотеку litellm та модулі DSPy для маршрутизації запитів до LLM. У нових версіях STORM було додано нативну підтримку DeepSeekModel. Для роботи в середовищі Antigravity 2.0 необхідно розширити базовий файл конфігурації.  

Змінні середовища:

Code snippet
DEEPSEEK_API_KEY="your_deepseek_api_key"
DEEPSEEK_API_BASE="https://api.deepseek.com/v1"
При ініціалізації класу STORMWikiLMConfigs необхідно перевизначити всі три основні моделі (дослідження, структурування, генерація) на об'єкти DeepSeek:

Python
# Ініціалізація моделей для конвеєра Antigravity 2.0
deepseek_kwargs = {
    "api_key": os.getenv("DEEPSEEK_API_KEY"),
    "api_base": os.getenv("DEEPSEEK_API_BASE"),
    "temperature": 1.0,
    "max_tokens": 8000
}

# Використання єдиної thinking-моделі для всіх етапів
lm_configs = STORMWikiLMConfigs()
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
Моделі deepseek-v4-pro виводять свій внутрішній процес мислення у форматі <think>...</think> перед основною відповіддю. Це гарантовано ламає стандартні регулярні вирази ChatAdapter або JSONAdapter у бібліотеці DSPy, яка лежить в основі STORM.
Для Antigravity 2.0 потрібно впровадити спеціалізований перехоплювач на рівні DSPy-адаптера:  

Створити або налаштувати тип dspy.Reasoning, який повідомляє адаптеру, що міркування слід отримувати з відповідного поля або парсити, ігноруючи блок <think> при валідації JSON чи розмітки.  

Увімкнути резервний механізм use_json_adapter_fallback=True у налаштуваннях DSPy. Якщо модель порушить формат під час генерації кінцевого виводу, адаптер автоматично повторить запит із жорсткішим примусом до JSON.  

4. Заміна базових DSPy-сигнатур STORM (Суворе якорування)
Щоб система працювала як універсальний middleware, а не генератор вікі-статей, необхідно переписати оригінальні класи спадкоємців dspy.Signature.

У файлі persona_generator.py скасувати виклик методу create_writer_with_persona, який шукає пов'язані теми та генерує ролі на кшталт "Wikipedia Writer". Замінити його на функціональний класифікатор компетенцій на основі вашої змінної main_question.  

У файлі outline_generator.py (клас WritePageOutline) вилучити вимогу до форматування заголовків символами #. Оригінальна функція clean_up_outline() схильна до збоїв парсингу. Її необхідно переписати під обробку суворого JSON-виводу логічної структури відповіді.  

5. Налаштування ретриверів (Retrievers)
Оскільки deepseek-v4-pro здатен здійснювати глибокий аналіз складних даних, якість подачі інформації через модулі knowledge_storm/rm.py є критичною.

При роботі із зовнішніми даними підключіть YouRM або DuckDuckGoSearchRM, але врахуйте необхідність обробки помилок парсингу API (наприклад, відловлювання AssertionError: keywords is mandatory для DuckDuckGo).  

При роботі з корпоративними даними Antigravity 2.0 ініціалізуйте VectorRM для завантаження документів, гарантуючи, що кожному документу присвоєно унікальний url, інакше система відкине їх під час агрегації. Обов'язково використовуйте аргумент для оновлення векторної бази (наприклад, --update-vector-store), щоб система мала доступ до актуального контексту.


2. Порівняльний аналіз провайдерів LLM
Система підтримує два взаємозамінні канали роботи з LLM, вибір між якими регулюється змінною оточення RND_LLM_PROVIDER у файлі 


