# ADR-0004. Passkey: webauthn-rs, discoverable, UV, amr по флагу BE

Статус: предложено, 15.10.2026.

## Контекст

Passkey — основной и в v1 единственный способ входа. Нужны: вход без логина, автоподстановка (conditional UI),
синхронизация iCloud / Google, несколько passkey на аккаунт, признак сильного входа в `amr` (RFC 8176) —
`mfa` вместе с `hwk`/`swk`. Аттестацию задача не требует.

## Решение

- `webauthn-rs` 0.5, Passkey API (UV обязательна, аттестация `none`), флаги `conditional-ui` (discoverable
  authentication) и `danger-allow-state-serialisation` (состояние ceremony сохраняется в БД на 5 минут; в cookie —
  только случайный id; так ceremony переживает перезапуск и не раздувает cookie).
- Регистрация требует resident key (discoverable). `user.id` — случайный handle аккаунта (64 байта), не `sub`.
  `excludeCredentials` — уже привязанные passkey аккаунта.
- Вход — `start_discoverable_authentication` → аккаунт по `userHandle` → проверка подписи.
- `amr`: UV + BE = 1 → `["swk", "mfa"]`; UV + BE = 0 → `["hwk", "mfa"]`. Флаги BE/BS сохраняются при регистрации и
  обновляются при входе.
- Название passkey — по AAGUID из открытого списка `passkeydeveloper/passkey-authenticator-aaguids` (MIT, вшит в
  бинарь, обновляется Renovate или вручную); пользователь может переименовать.
- WebAuthn Signal API (`signalUnknownCredential`, `signalAllAcceptedCredentials`, `signalCurrentUserDetails`) — как
  улучшение там, где браузер его поддерживает.
- Перед этапом 1 — сверить версию `webauthn-rs`, имена флагов и поведение conditional UI в Safari / Chrome.

## Последствия

- `webauthn-rs` зависит от OpenSSL: в образе — vendored-сборка или `libssl` из базового образа (сравнить на
  этапе 0).
- Без аттестации `hwk`/`swk` — эвристика по флагу BE; это документируется в `docs/integrate.md`.
- Синхронизируемый passkey вытащить с устройства сложнее, чем пароль, но он живёт в облачном аккаунте Apple /
  Google: компрометация этого аккаунта — риск, который пользователь несёт и в других сервисах.

## Отвергнутые варианты

- Запрашивать аттестацию — лишние диалоги у пользователя, а синхронизируемые passkey её всё равно не дают.
- Другие крейты (`webauthn-rp` и др.) — задача прямо называет `webauthn-rs`; он самый зрелый для серверной стороны.
