# Lesson 9 — Security audit

Дата: **15 жовтня 2026 року**. Scope: auth, middleware, book/loan/user handlers,
repositories, routes, SQL migrations and dependency graph. Review specification
is recorded in `prompts.txt` (Task 3.1).

Метод: перевірка коду й маршрутів, зіставлення з початковим імпортом `791f07e`,
негативні HTTP-тести проти справжнього router/PostgreSQL, конкурентні запити,
SQL rollback/up, race detector та govulncheck. Локальні результати — у
`evidence/verification.txt`. Рядки нижче стосуються файлів цієї ревізії;
для вихідного стану вказано commit.

Severity у False Positive означає потенційний вплив **якби припущення
підтвердилося**. False Positive не означає виправлену вразливість. Не всі десять
перевірених припущень є реальними дефектами.

| ID | Severity | Місце | Знахідка / triage | Статус і перевірка |
|---|---|---|---|---|
| AUTH-01 | High | `791f07e:internal/app/app.go:26`; тепер `internal/app/app.go:48` | Успадковані POST/PUT/DELETE каталогу були публічними. Неавтентифікований клієнт міг змінити або видалити книги. | **Fixed Now** — protected → staff/admin groups; TestRBAC_Matrix перевіряє 401/403/201, оновлення та видалення, і відсутність запису від denied handlers. |
| DEP-01 | High | `791f07e:go.mod:3`; тепер `go.mod:3` і dependency versions | Початковий Go 1.27.1 та старі pgx/x/net містили відомі вразливості. govulncheck вказав Go 1.27.2, pgx 5.9.2 і x/net 0.60.0 як виправлені версії. Серед прикладів — GO-2026-6603, GO-2026-6617, GO-2026-5004. | **Fixed Now** — оновлено toolchain і сумісні транзитивні залежності; повторний govulncheck: 0 affected vulnerabilities, 0 imported-package findings. Це результат сканера, не твердження, що кожен початковий advisory був експлуатований у нашому API. |
| AUTH-02 | Medium | `internal/auth/jwt.go:29`, `internal/middleware/auth.go:13` | Немає per-token logout/revocation. Викрадений Bearer token може діяти до exp, поки обліковий запис і роль не змінилися. | **Tracked for Later** — додати jti/token version або server-side revocation і refresh rotation. Поки що роль із БД перевіряється на кожному запиті; TestRoleChangeInvalidatesOldToken доводить відкликання старих привілеїв. |
| RATE-01 | Medium | `internal/middleware/rate_limit.go:13` | Ліміт спроб входу process-local: після restart скидається, між кількома інстансами не спільний. | **Tracked for Later** — перед multi-instance deployment потрібен shared limiter. Локальний ліміт 20/min/IP має bounded storage; forwarded IP не довірений. TestAuthRateLimitCannotBeBypassedWithForwardedIP перевіряє 429 і abort. |
| DATA-01 | High | `internal/model/user.go:10`, `internal/repository/user_repository.go:38` | Припущення: повернення User відкриває password hash. У фактичній моделі PasswordHash має json:"-", список users додатково виключає його із SELECT. | **False Positive** — TestUsersResponseNeverContainsPasswordHash і реєстраційні тести перевіряють JSON та серіалізацію моделі. Захист має залишатися при додаванні нових DTO. |
| DATA-02 | High | `internal/model/security.go:13`, `internal/handler/errors.go:53` | Припущення: роль/ID/hash можна призначити через register або profile. DTO не містять цих полів, decoder відхиляє невідомі поля, registration явно задає reader. | **False Positive** — TestAuthentication_TableDriven та TestProfileCannotEscalatePrivileges повертають 422 і перевіряють збережену роль. |
| INJ-01 | High | `internal/repository/user_repository.go:28`, `internal/testutil/database.go:62` | Припущення: lookup email або SQL concatenation дозволяють injection. Email передається параметром; конкатенація назви схеми є лише в test helper і використовує випадковий hex, не HTTP input. | **False Positive** — TestParameterizedEmailLookup не знаходить користувача через injection string і підтверджує незмінну кількість записів. Book/loan queries також використовують placeholders. |
| LOG-01 | Medium | `internal/middleware/middleware.go:26`, `internal/repository/db.go:18` | Припущення: logger записує пароль, JWT або email із запиту. Logger записує лише method/path/status/duration, без body/headers/query; GORM має ParameterizedQueries=true. | **False Positive** — TestLoggingSkipsCredentials перевіряє body, header, query і email markers. Не додавати body logging без окремого redaction. |
| JWT-01 | High | `internal/auth/jwt.go:40` | Припущення за навчальним mock-прикладом: parser приймає довільний HMAC algorithm або токен без exp. Production parser pin-ить HS256 та вимагає expiration і валідну typed identity. | **False Positive** — TestAuthMiddleware_RejectsInvalidTokens відхиляє HS384, alg:none, no-exp, expired, wrong signature, zero ID, unknown role та forged elevated role. Mock middleware із ZIP не включено в production. |
| DEP-02 | Low | `go.mod` → `golang.org/x/crypto`; неімпортований `openpgp` | govulncheck повідомляє GO-2026-5932 для неімпортованого OpenPGP у модулі, який потрібен для bcrypt. Сам advisory реальний; код застосунку не імпортує цей пакет. | **False Positive** для застосовності до API — verbose scan підтвердив 0 package/symbol vulnerabilities. Не використовувати deprecated OpenPGP; не називати цей модуль повністю вільним від advisories. |

## Додатково перевірено

- `/me/loans` використовує лише verified user_id; довільний user_id filter відхиляється.
- `TestLoansOwnershipAndLifecycle` показує, що reader бачить тільки власну видачу.
- Concurrent registration: один успіх, один conflict для email різного регістру.
- Concurrent borrowing: один успіх, один conflict, рівно одна активна видача на copy.
- Повторне повернення → 409; reader return → 403; copy стає available атомарно.
- Auth migration rollback зберігає успадковане role; повний up/down/up зберігає нормалізацію.
- DTO/body validation обмежує розмір до 1 MiB. bcrypt default cost=10 і byte limit=72.
- Login errors однакові для unknown/wrong credentials; відсутній account проходить dummy comparison.
- CORS не замінює авторизацію. Bearer-захист перевіряється middleware незалежно від Origin.

## Межі перевірки

Автоматичний аудит не гарантує відсутність усіх вразливостей. Локальний coursework
API працює на loopback; публічне deployment, TLS/proxy configuration, distributed
rate limiting і token revocation потребують окремої роботи. Повторювати dependency
scan перед наступною публікацією. Результати тут описують саме перевірену ревізію,
а не майбутні зміни.
