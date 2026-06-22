# Технічна пояснювальна записка для LLM-агента-реалізатора: PerspectaMind Framework1.


## Що таке PerspectaMind (основний бекграунд)PerspectaMind — це система від Stanford OVAL Lab (Open Virtual Assistant Lab), опублікована на NAACL 2024.
Повна назва: Synthesis of Topic Outlines through Retrieval and Multi-perspective Question Asking.Мета: Автоматизувати етап «pre-writing research» — створення якісного структурованого outline + контенту Wikipedia-подібної якості з нуля, з використанням пошуку в інтернеті та мульти-перспективного підходу. Система суттєво зменшує галюцинації та покращує організацію та широту покриття порівняно зі звичайними RAG/outlines-based методами (на 25% краща організація за оцінками Wikipedia-редакторів). 

Ключова ідея: замість прямого запиту LLM «напиши статтю про X», система імітує роботу дослідника:Виявляє різноманітні перспективи (perspectives).
Симулює «розмови» між агентами з різними поглядами та topic expert.
Збирає grounded інформацію через web search.
Створює curated outline.
Генерує повну статтю з цитатами.

2. Основні етапи оригінальної архітектури (з паперу)Perspective Discovery — Аналіз схожих тем/Wikipedia-статей для виявлення різних поглядів (практик, академік, скептик, економіст тощо).
Perspective-Guided Question Asking — Симуляція розмов: кожен perspective-agent ставить питання topic-expert, який відповідає на основі trusted web sources.
Information Curation — Збір, deduplication, мапінг суперечностей.
Outline Generation + Article Writing — Створення coherent структури та повного тексту.

Є також Co-PerspectaMind — розширення з human-in-the-loop collaboration. Основні джерела (обов’язково вивчити):Головний папір (PerspectaMind): https://arxiv.org/abs/2402.14207 (або PDF: https://arxiv.org/pdf/2402.14207.pdf) 

GitHub репозиторій (офіційна реалізація): https://github.com/alenev/perspecta_mind 

Демо / Research Preview: https://perspecta_mind.genie.stanford.edu (використовує GPT-4o-mini з жорстким фільтруванням контенту та обмеженнями на arbitrary instructions — саме тому потрібно будувати свій SaaS). 

Co-PerspectaMind папір: https://arxiv.org/abs/2408.15232
Офіційний сайт проєкту: https://perspecta_mind-project.stanford.edu/

3. Популярна промпт-адаптація (для Claude та інших LLM)Багато розробників (включаючи тред @heynavtoor
) адаптували ключові ідеї PerspectaMind у 4-промптовий workflow без повноцінного retrieval-пайплайну:Multi-Perspective Scan — Генерація 5+ expert perspectives.
Contradiction Map — Мапування суперечностей, згод та прогалин.
Synthesis & Briefing — Інтеграція в coherent research briefing з рекомендаціями.
Peer Review / Self-Critique — Перевірка на bias, слабкі місця, надійність.

Цей lightweight-варіант добре працює як middleware у чат-додатку.4. Рекомендації для реалізації в SaaS (middleware)Модель: Використовуйте thinking-моделі (Claude Sonnet/Opus 4, o1-like, Grok thinking) для кращого multi-step reasoning.
Retrieval: Обов’язково додавайте browsing/search (Perplexity-style, Tavily, Serper, Exa тощо) на етапах question-answering.
Архітектура: LangGraph / CrewAI / AutoGen для оркестрації агентів (Perspective Agents + Writer Agent + Critic Agent).
Output: Wikipedia-style article + citations + confidence scores + contradiction map + actionable insights.
Обмеження оригінального демо: Жорстке фільтрування (не дозволяє custom instructions, блокує sensitive теми) — тому самостійна реалізація потрібна для комерційного продукту.

5. Додаткові корисні матеріалиОгляд архітектури в GitHub README.
Приклади промптів з треду heynavtoor (можна шукати за назвою).
Порівняння з Perplexity / Google Deep Research (PerspectaMind часто виграє в організації та широті).

Завдання для агента-реалізатора:
Вивчити папір https://arxiv.org/pdf/2402.14207.pdf + GitHub, потім спроектувати scalable multi-agent pipeline, який реалізує core PerspectaMind-логіку як middleware між користувачем і thinking-LLM, з надійним retrieval та fact-grounding.


## Приклад аналізу
Потрібно приблизно сформувати розуміння процессу використання perspecta_mind в аналізі пошуку можливостей збільшення прибутків послуги надання системного адміністрування мереж. Це середній бізнес. Адміністратори системні налаштовують віддалене керування мережевою інфраструктурою кампаній. Мені потрібно розуміти мій шлях аналітика-дослідника можливостей збільшення прибутків цього сервісу. Яку мені інформацію потрібно запитувати, які основні питання аналізувати в PerspectaMind. Підготуй для мені структуровану інструкцію.

