# Мультиагентное сотрудничество и коэволюция

Глава 10 рассматривает multi-agent не как автоматическое умножение интеллекта, а как архитектурный выбор с измеримой ценой. Система выигрывает, когда разделяет контексты, вводит новые внешние сигналы, параллелит действительно независимую работу или закрепляет разные контракты ответственности. Послесловие расширяет этот взгляд: реальные задачи обучают harness, его исправления становятся данными для следующей модели, а новая модель сдвигает harness к следующей ненадёжной границе.

## 1. Shared Context против Context Isolation

Первая ось архитектуры — видят ли агенты одну траекторию. Shared context не отбрасывает записи при передаче: новая роль получает всю историю, хотя system prompt и tools меняются. Это не гарантирует, что модель функционально использует каждую деталь: длинный контекст всё равно может вытеснить её из внимания. Такой режим удобен для короткой последовательности «уточнение требований → реализация → review», но переносит инерцию рассуждения прежней роли.

Non-shared context даёт каждому агенту собственную историю и состояние. Информация передаётся явно через typed parameters, files или messages, поэтому компоненты проще тестировать, изолировать и запускать параллельно. Книжная эвристика: если совокупный контекст превысит около половины окна, лучше не делиться; если для корректности недопустима потеря детали — делиться. На практике ранние стадии могут наследовать историю, а после saturation переходить к distilled handoff. **Экстраполяция анализа:** решение о переключении стоит принимать по измеренным context occupancy, retrieval miss и handoff-loss rate, а не только по фиксированным 50%.

## 2. Топологии: peer, manager и decentralized

Peer collaboration подходит 2–3 равноправным ролям и короткому iterative loop. Proposer пишет, Reviewer получает render, tests или tool evidence и возвращает структурированные замечания. Manager pattern, по эвристике главы, уместен при более чем пяти подзадачах, динамическом планировании или сложных зависимостях: Manager декомпозирует, выбирает исполнителей, следит за progress, исправляет план и синтезирует результат. Специализированный агент представлен для него как tool; обратно возвращаются conclusion, findings и artifact paths, а не полная trajectory. В planner–executor задачах, где ошибку декомпозиции трудно компенсировать, сильнейшую модель и лучший prompt стоит отдать Planner/Manager.

Decentralized pattern убирает runtime-координатора. Handoff package содержит задачу и acceptance criteria, подтверждённые ограничения и ссылки на artifacts. MetaGPT демонстрирует SOP и subscription message pool, AutoGen — гибрид shared conversation с централизованным speaker selector, Swarm — настоящий peer-to-peer handoff. **Экстраполяция анализа:** topology следует фиксировать в machine-readable graph с ownership, allowed transitions и termination conditions, чтобы обнаруживать циклы и «ничейные» задачи.

## 3. Критерий нового сигнала и экономика compute

Проектная эвристика книги: появляется ли информация, которой не было при первой генерации? Повторное чтение того же текста и debate при равном thinking budget часто не превосходят одного агента. Напротив, compiler error, test result, screenshot, database query или независимый source check меняют epistemic state. Поэтому Reviewer ценен не отдельной «личностью», а доступом к новому каналу наблюдения. На WebGen-Bench система WebGen-Agent повысила результат Claude 3.5 Sonnet с 26,4% до 51,9% благодаря visual feedback.

Больше steps также не гарантируют глубины: без budget awareness агент рано насыщается поверхностным поиском. Manager должен распределять бюджет по сложности, сначала исследовать широко, затем переходить к exploitation, тестированию и refinement. В данных Anthropic их multi-agent research system использовала около 15× токенов относительно chat-взаимодействия; на BrowseComp token usage объяснял около 80% дисперсии результата. **Экстраполяция анализа:** сравнивать архитектуры по quality per dollar, latency percentile и marginal gain на дополнительный agent-step.

## 4. Data Plane, Control Plane и A2A

Без общего контекста collaboration держится на двух плоскостях. Data plane — virtual filesystem из private scratchpads, persistent shared workspace, permission-bound external mounts и read-only system resources. Передача path вместо content экономит контекст. Для concurrent coding лучше отдельные Git worktrees и явная merge boundary.

Control plane включает structured message envelope, status state machine и termination. Pull status прост, push своевременнее; timeout и heartbeat страхуют оба. При «один нашёл — остальные остановились» Manager рассылает graceful terminate, ждёт acknowledgments и лишь затем применяет forced fallback. Lock или idempotent settlement устраняет гонку двух почти одновременных успехов. Между организациями A2A добавляет Agent Card, task lifecycle и opaque artifact exchange: MCP связывает agent с tool, A2A — agent с agent. **Экстраполяция анализа:** все сообщения и transitions следует писать в append-only trace с correlation ID для replay и аудита.

## 5. Конфликты и каскадное усиление ошибок

MAST группирует сбои в Specification and System Design Failures, Inter-Agent Misalignment и Task Verification and Termination. На файловом уровне два writers создают lost update; optimistic lock сравнивает version при записи. Но cross-file semantic conflict не виден Git: один агент перенумеровал иллюстрации, другой сохранил старые ссылки. Здесь нужны dependency-aware scheduling и global consistency check.

Опаснее semantic cascade. В примере Terminology Agent неудачно сводит разные понятия reasoning и inference к одному переводу; Translation Agents воспроизводят решение, а Proofreader принимает единообразие за доказательство правильности. Ошибка получает авторитет через повторение. Chain breaker — независимый Reviewer, который не наследует рассуждение, а сверяет original evidence с final conclusion; для high-risk решений используются tests, compiler и database truth. **Экстраполяция анализа:** provenance graph должен показывать, какие outputs зависят от одного исходного claim, чтобы одной invalidation пометить весь каскад.

## 6. Agent Society и коэволюция модели с Harness

При длительном свободном взаимодействии возникают незапрограммированные коллективные эффекты. В Stanford AI Town 25 агентов с memory, reflection и planning распространили приглашение и согласовали вечеринку без центрального сценария. Книга приводит Moltbook как сообщаемый пример агентной сетевой культуры, а в описанных в ней запусках Vending-Bench Arena — ценовые войны и предложения о согласовании цен; масштаб и степень автономности таких явлений требуют отдельной проверки. Experiment 10-8 предлагает проект Voice Werewolf с code-driven Judge и ролевым контекстом, но результатов не сообщает.

Послесловие связывает это с model–agent co-evolution. Агентная обвязка (Harness) фиксирует места, где модель ненадёжна: compression, retries, permissions, circuit breakers. Реальные failures превращают patches в training signals; следующая модель internalizes часть ограничений, после чего Harness удаляет этот слой и перемещается к новой frontier. Послесловие ссылается на LangChain Deep Agents: изменение Harness при фиксированной модели подняло результат Terminal-Bench 2.0 с 52,8% до 66,5% (+13,7 п.п.). **Экстраполяция анализа:** поддерживать карту «временное ограничение Harness → подтверждающий eval → кандидат на model training → критерий безопасного удаления» и не считать исчезновение одного слоя концом инженерной работы.
