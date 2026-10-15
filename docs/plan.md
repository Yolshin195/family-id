# План по этапам

К [ТЗ](spec.md). Каждый шаг — один или несколько маленьких коммитов с тестами. Шаг считается сделанным при зелёных
`cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`,
`cargo deny check`. Всё, что зависит от внешних сервисов и версий, сверяется с документацией в начале этапа.

## Этап 0 — основа (после одобрения ТЗ)

| # | Шаг | Результат |
| --- | --- | --- |
| 0.1 | `arch.py init --name family_id --title "family-id" --table-prefix fid_` (эталон v2), `LICENSE-MIT`, `LICENSE-APACHE` | Пустой workspace собирается |
| 0.2 | Kernel: `config` (TOML + env + `*_FILE`, проверки продакшена), `crypto`, `i18n`, `request`, `throttle`, `audit` (порт), заготовка `web` | Юнит-тесты конфигурации и проверок продакшена |
| 0.3 | `deny.toml`, `renovate.json`, `.github/workflows/ci.yml` (fmt, clippy, test с PostgreSQL, deny, audit) | CI зелёный |
| 0.4 | `Dockerfile` (multi-stage, non-root, read-only), сравнение OpenSSL vendored / системная, workflow публикации amd64/arm64 в GHCR | Образ собирается, размер и Trivy зафиксированы |
| 0.5 | `compose.yaml` (PostgreSQL 16 + family-id), `Makefile`, `.env.example`, `README.md` | `make` поднимает стенд, `/healthz` отвечает |

## Этап 1 — OIDC-провайдер и passkey

Начало этапа: сверить версии `webauthn-rs`, `jsonwebtoken`, `openidconnect`, `askama`, флаги conditional UI;
текст RFC 9700 (номера разделов); текущую версию Conformance Suite и наличие PKCE в плане Basic OP.

| # | Шаг | Результат |
| --- | --- | --- |
| 1.1 | Фича `keys`: генерация Ed25519 и RSA, шифрование мастер-ключом, JWKS, подпись JWT, ротация, `keys rotate` | Тесты: подпись и проверка, ротация с перекрытием, отказ без мастер-ключа |
| 1.2 | Фича `audit`: запись, чтение по аккаунту, чистка старше года | Тесты на memory и PostgreSQL |
| 1.3 | Фича `accounts`: создание, профиль, имя пользователя, статус | Тесты правил имени и уникальности |
| 1.4 | Фича `sessions`: SSO-сессии, потоки входа, свежая аутентификация, список устройств, выход на устройстве / на остальных / везде | Тесты сроков, хешей, отзыва |
| 1.5 | Фича `passkeys`: регистрация, discoverable-вход, повторная аутентификация, `amr` по флагу BE, названия по AAGUID | Тесты ceremony на зафиксированных ответах аутентификатора |
| 1.6 | Kernel `web` + фича `portal`: каркас, дизайн-токены и шрифты из бухгалтерии, `webauthn.js`, страницы входа и регистрации на трёх языках | E2E Playwright с виртуальным аутентификатором: регистрация и вход |
| 1.7 | Фича `oidc`: клиенты из конфигурации, discovery, `/authorize` (`prompt`, `max_age`, `id_token_hint`, `ui_locales`), `/token` (код + PKCE, refresh с ротацией), `/userinfo`, `/revoke`, `/introspect`, согласие для чужих клиентов, `iss` в ответах | Интеграционные тесты протокола |
| 1.8 | Тесты RFC 9700 (по тесту на атаку) | `crates/features/oidc/tests/rfc9700_*.rs` |
| 1.9 | Кабинет: профиль, способы входа, устройства и сессии, журнал | E2E: добавить и удалить passkey, выйти на другом устройстве |
| 1.10 | Демо-клиент (минимальный RP, два экземпляра в compose), E2E «вход → второй клиент без нажатий» | E2E зелёный |
| 1.11 | Сборка `conformance`, `compose.conformance.yaml`, Suite в CI: Basic OP, Config OP | Профили проходят |
| 1.12 | `docs/security.md` (ASVS L2), `docs/deploy.md`, `docs/integrate.md` (v1: подключение OIDC-клиента), бэкапы `make backup` / `make restore-check` | Документация и проверка восстановления |
| 1.13 | Проверка памяти (< 64 МБ), `/security-review`, ручная приёмка владельцем на MacBook и iPhone | Готово к продакшену v1 |

## Этап 1.1 — Telegram, единый выход, восстановление

Начало этапа: открыть доступ к `core.telegram.org` / `oauth.telegram.org` или сверить документацию Telegram Login
иначе; проверить claims, алгоритмы подписи, настройку @BotFather → Web Login.

| # | Шаг | Результат |
| --- | --- | --- |
| 1.1.1 | Фича `upstream`: порт `UpstreamProvider`, generic OIDC-адаптер, провайдер Telegram, связанные учётки | Тесты проверки ID-токена Telegram на фикстурах |
| 1.1.2 | Вход, регистрация и привязка через Telegram в `portal`, предложение второго способа | E2E с поддельным OIDC-провайдером |
| 1.1.3 | Outbox, back-channel logout, RP-Initiated Logout (`/logout`) | Интеграционные тесты доставки и повторов |
| 1.1.4 | SET `account-purged`, удаление аккаунта, блокировка, ссылка восстановления (CLI) и сессия восстановления | Тесты и E2E |
| 1.1.5 | «Подключённые приложения» в кабинете | E2E: отзыв |
| 1.1.6 | Conformance: RP-Initiated Logout, Back-Channel Logout; E2E «выйти везде» закрывает сессии обоих демо-клиентов | Профили проходят |
| 1.1.7 | `docs/integrate.md`: logout, SET, `prompt=none`; раздел «что поменять в бухгалтерии»; `/security-review` | Готово к продакшену v1.1 |

## Этап 1.2 — OAuth 2.1 для MCP

Начало этапа: сверить спецификацию MCP (актуальная версия), claude.com/docs/connectors, CIMD Claude и Claude Code.

| # | Шаг | Результат |
| --- | --- | --- |
| 1.2.1 | Фича `mcp_clients`: CIMD (белый список, SSRF-защита, кэш), предзаведённые публичные клиенты, реестр ресурсов | Тесты: редирект, размер, таймаут, частный IP, несовпадение `client_id` |
| 1.2.2 | `resource` (RFC 8707) в `/authorize`, `/token`, refresh; `aud`; интроспекция только владельцу ресурса | Тесты: токен ресурса A у ресурса B |
| 1.2.3 | Делегированное согласие: `consent_challenge`, API `/api/consent/*`, `consent_verifier`, `remember`; Swagger для `/api` | Интеграционные тесты, E2E с демо-ресурсом |
| 1.2.4 | Кабинет: ассистенты и их гранты, отзыв | E2E |
| 1.2.5 | `docs/integrate.md`: MCP-ресурс (метаданные RFC 9728, `401`, интроспекция, страница согласия с чек-листом) | Документация |
| 1.2.6 | Подключение Claude к демо-ресурсу, затем к бухгалтерии (после её доработки); `/security-review` | Claude работает с инструментами |

## Этап 2 — по отдельному решению

Код на email (SMTP), Google, LINE, Apple (generic OIDC), GitHub, Facebook (OAuth2), админ-страницы.
