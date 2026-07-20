# Оценивание, наблюдаемость и научная итерация

Источник: Bojie Li, *AI Agents in Depth: Design Principles and Engineering Practice*, Chapter 6 “Evaluating Agents”, 2026. AndroidWorld baseline и assumed results H1/H3/H4/H5/H6 прямо обозначены как гипотетический учебный пример; H2 не имеет численного результата, а H7/H8 остаются непроверенными гипотезами следующего цикла. Ничего из этого нельзя представлять как опубликованный benchmark result.

Глава отвечает не на вопрос «какая модель сильнее вообще», а на более инженерный: как доказать, что конкретный агент действительно стал лучше. Объектом проверки служит связка model + Harness. Если при неизменном Harness более сильная модель не улучшает результат, вероятен системный bottleneck; если score резко меняется со сменой модели, ограничение связано с model capability или чрезмерной зависимостью Harness от нее. Ablation отвечает на другой вопрос: какой именно компонент Harness создает ценность.

## Среда как исполняемый контракт

Evaluation environment фиксирует dataset, initial state, tools, rubric и interaction protocol. Для tool-calling задач подходят executable verifiers: тесты, состояние базы, наличие артефакта. Для общения нужен user simulator с Progressive Information Disclosure. В τ-bench пользователь не сообщает все сразу, а τ²-bench позволяет ему менять общую среду: например, включить airplane mode, после чего агент обязан наблюдать новый state. Проверять следует и outcome, и trajectory: запись в базе доказывает действие, диалог — корректность подтверждения и объяснения.

## Dataset должен диагностировать

Хороший набор балансирует clarity и openness, authenticity и controllability, coverage и cost. SWE-Bench Verified использует FAIL_TO_PASS и PASS_TO_PASS; AndroidWorld параметризует задачи и проверяет final UI state; Terminal-Bench объединяет container state с функциональным запуском; τ²-bench отделяет known information от сценария раскрытия. Capability tags, уровни сложности, trap tasks и edge cases превращают общий score в профиль причин. Сам benchmark тоже версия продукта: OSWorld-Verified пришлось чинить environment, descriptions, verification logic и initial states. Конкретные evaluation tasks нельзя возвращать в training set, иначе измеряется запоминание.

## Метрики и статистика

Process metrics включают legality/correctness tool calls, path efficiency, retrieval coverage, tokens, latency и cost; outcome metrics — task success, quality, safety и robustness. `Pass@k` означает хотя бы один успех, `Pass^k` — все k успехов, `Best@k` — лучший score. При `Pass@1 = 60%` пять попыток дают `Pass@5` около 99%, но `Pass^5` лишь около 7,8%: capability ceiling высок, reliability низка.

Один прогон не доказывает улучшение. Для 100 случаев при success rate 70% standard error около 4,6%, поэтому разница 73% против 70% тонет в шуме. Нужны несколько seeds, mean и spread. Если конфигурации запущены на одинаковых задачах, полезнее paired wins/losses и McNemar’s test. При параллельной проверке многих гипотез растет false-positive risk; глава предлагает ужесточать threshold или независимо воспроизводить положительный результат.

## Автоматический судья тоже требует оценки

LLM-as-a-Judge работает только поверх экспертной rubric с конкретными уровнями, pitfalls и veto. Hallucination или serious safety violation не компенсируются стилем. Judge подвержен length bias, drift и reward hacking, поэтому его калибруют на human gold set, измеряют agreement или Cohen’s kappa и повторяют калибровку после смены модели или rubric. Heterogeneous judges снижают same-family bias. В pairwise comparison позиционный перекос контролируют двойной оценкой A/B и B/A; Elo или Bradley–Terry затем агрегирует сравнения в относительный ranking.

## Observability замыкает контур

Один trace состоит из spans для LLM, tools и retrieval с inputs/outputs, timing, tokens, cost и errors. OpenTelemetry задает общий span model, OpenInference — LLM semantics. Трасса нужна не только для debugging: она показывает повторные вызовы, дорогие tool results и пустой retrieval. При падении score сначала проверяют environment и scorer — resource exhaustion и сломанный verifier внешне неотличимы от model regression. Production failures после sanitization становятся regression cases, поэтому dataset постепенно приближается к реальному распределению.

## От отчета к эксперименту и тренировочной арене

Встроенная evaluation infrastructure требует independent ablation switches, multi-armed A/B tests, mechanism/target/guardrail metrics, feature flags и versioned prompt snapshots. Model selection учитывает не только success, но TTFT, p95, thinking tokens, tool fees и cost per successful task. Гипотетический AndroidWorld case показывает логику решения: не включать thinking глобально ради небольшой группы counting tasks, а проверить conditional routing.

Наконец, evaluation environment может стать simulation environment. Verifier превращается в RL reward, однако обучение требует миллионов interactions, чистого deterministic reset, randomization и высокой parallel throughput. AWorld иллюстрирует цифровой sandbox, RoboTwin2 — embodied physics и domain randomization.

Экстраполяция для production-команд: унифицированный `reset/step/observe/verify` interface позволяет повторно использовать одну механику в CI, offline evaluation и training, сохраняя закрытые evaluation instances изолированными от обучения. Автоматическое превращение подозрительного trace в draft regression case полезно пропускать через human triage, чтобы случайный шум не стал частью продуктовой спецификации.
