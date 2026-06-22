## А. Інтеграція через DeepSeek API (REST HTTP)
Функція реалізації: 
call_deepseek
Механізм роботи: Використовує асинхронний Rust-клієнт reqwest для виконання HTTP POST запитів до офіційного ендпоінту https://api.deepseek.com/v1/chat/completions.
Авторизація: Зчитує DEEPSEEK_API_KEY із системного оточення.
Мапінг моделей: За замовчуванням використовуються моделі deepseek-v4-pro (для синтезу та архітектури) та deepseek-v4-flash (для критики та аудиту). За потреби їх можна перевизначити через змінні оточення (наприклад, RND_SKEPTIC_MODEL_DEEPSEEK).
Облік токенів: Отримує точні показники споживання безпосередньо з JSON-відповіді сервера (usage.prompt_tokens + usage.completion_tokens) та логує їх у файл logs/rnd_deepseek_token_usage.log.

## Б. Інтеграція через Antigravity CLI (agy)
Функція реалізації: 
call_antigravity_cli
Механізм роботи: Запускає локальну CLI-утиліту agy через std::process::Command всередині віртуального терміналу script -q -c (для уникнення проблем з буферизацією введення-виведення).
Команда запуску:
bash
/home/lden_eons_28cores/.local/bin/agy --model <model> --dangerously-skip-permissions -p '<system_prompt + prompt>'
(Спеціальні символи та одинарні лапки в промпті автоматично екрануються).
Мапінг моделей: Визначається через змінні RND_<ROLE>_MODEL_ANTIGRAVITY. За замовчуванням використовуються моделі лінійки gemini-3.5-flash-low, gemini-3.5-flash-medium та gemini-3.1-pro-high / gemini-3.1-pro-low.
Облік токенів: Оскільки CLI-інтерфейс не повертає метаданих API, споживання оцінюється локально за допомогою бібліотеки tiktoken_rs (кодування cl100k_base). Для української кирилиці реалізовано резервний евристичний алгоритм: (кількість слів * 2.5). Дані логуються у файл logs/rnd_antigravity_token_usage.log.