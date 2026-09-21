# TransFin — операційна частина (приватно)

**Оновлено:** 21.09.2026 (Київ), сесія ERP-8, P8.
**Що тут:** усе, що стосується конкретних машин, доступів, секретів, тимчасових заходів та інцидентів. Знання про код — у `HANDOFF.md` репозиторію `transfin-backend`.
**Правило для цього файла:** **значень секретів тут немає й не буде** — лише де вони лежать, хто їх читає і як їх змінити.

---

## 1. Сервер (продакшн)

- **Адреса:** https://transfin.uk. DNS — Cloudflare (DNS-only).
- **Хостинг:** Hetzner CX23, `46.224.18.229`, Ubuntu 26.04. RAM 3,7 GiB, **swap немає**.
- **Вхід:** лише SSH за ключем, `root@46.224.18.229`, ключ `~/.ssh/id_ed25519` на Mac. Вхід за паролем вимкнено (`/etc/ssh/sshd_config.d/10-transfin-hardening.conf`).
- **Застосунок:** `transfin.service` (systemd, користувач `transfin`, `127.0.0.1:8000`, 1 worker); код у `/home/transfin/backend`, Python 3.12 через uv (`/opt/uv/python`). У unit-файлі `QUEUE_AUTOSTART_IN_WEB=false`, `QUEUE_POLLING_ENABLED=false` — сайт на сервері eCherha **не** читає.
- **e-TOLL recorder:** `transfin-etoll-recorder.service` (з `deploy/`), користувач `transfin`, логін бази `etoll_recorder`, `MemoryMax=256M`. Стан на 21.09: `active` + `enabled`.
- **Секрети:** `/etc/transfin/transfin.env` (`root:transfin`, `0640`). Ключі: `DATABASE_URL`, `LEDGER_DATABASE_URL`, `SESSION_SECRET_KEY` (64 символи), `SESSION_COOKIE_SECURE=true`, `LONTEX_ATLAS_BASE_URL/LOGIN/PASSWORD`, `ETOLL_DATABASE_URL`, `ALLOWED_HOSTS=transfin.uk,www.transfin.uk` (з 21.09). **Ніколи не друкувати.** Перед запуском коду застосунку на сервері завантажувати: `sudo -u transfin bash -c 'set -a; . /etc/transfin/transfin.env; set +a; …'` (значення не потрапляють в argv).
- **nginx + Let's Encrypt:** `/etc/nginx/sites-available/transfin`; http і www → https apex; невідомий Host → 444 або відмова TLS; `/docs`, `/redoc`, `/openapi.json` → 404; обмеження частоти на логін. Застосунок сам перевіряє Host (`ALLOWED_HOSTS`, з 21.09): **локальні перевірки лише з `-H 'Host: transfin.uk'`**, інакше 400. На сервері ніщо не звертається до `127.0.0.1:8000` напряму (перевірено 21.09: таймери, cron, `atq`, скрипти, агентів моніторингу немає).
- **PostgreSQL 18:**

  | База / роль | Для чого |
  |---|---|
  | `transfin_backend` (власник `transfin_backend`) | основна база сайту; схема `etoll` (власник `etoll_owner`) |
  | `transfin` (власник `transfin`) | Ledger; alembic з 21.09 — `0006_audit_actor` |
  | `transfin_worker` | воркер eCherha: лише DML, default privileges на нові таблиці; **без `UPDATE` на `crossing_log`** (тимчасово, §4) |
  | `etoll_owner` (NOLOGIN) | власник усього в `etoll`; члени з `INHERIT FALSE, SET TRUE`: `transfin_backend`, `etoll_recorder` |
  | `etoll_recorder` (LOGIN, SCRAM) | запис позицій; лише DML |

  CONNECT/TEMP для PUBLIC на `transfin` і `postgres` забрано.
- **Бекапи:** `transfin-pgdump.timer` щодня ~03:15 UTC → `/var/backups/transfin/{transfin_backend,transfin,globals}/`, 7 копій, скрипт `/usr/local/sbin/transfin-pgdump.sh` сам перевіряє дамп (`pg_restore --list`). **Копій поза сервером (off-site) немає.** Відкладені копії: перед міграцією e-TOLL — `/var/backups/transfin/pre-etoll-transfin_backend_20260920T173502Z.dump`; перед P1–P8 — `pre-p1p8-transfin_backend_20260921T080917Z.dump` і `pre-p1p8-transfin_20260921T080917Z.dump`. Для `pg_restore` від `postgres` дамп треба спершу скопіювати в теку, яку `postgres` може читати (тека бекапів — `root 0700`).
- **Користувач тунелю `transfin-tunnel`:** без shell і пароля; ключ дозволяє лише перенаправлення на `127.0.0.1:5432`; `/etc/ssh/sshd_config.d/20-transfin-tunnel.conf`.
- **Alembic:** жоден ланцюжок не будує схему з порожньої бази; сервер зібрано через `create_all` + `alembic stamp head` 16.09.

### Розгортання

```bash
cd /home/transfin/backend
sudo -u transfin git status --porcelain        # має бути порожньо
sudo -u transfin git fetch origin ui/journal-redesign
sudo -u transfin git log --oneline HEAD..origin/ui/journal-redesign
sudo -u transfin git merge --ff-only origin/ui/journal-redesign
systemctl restart transfin.service
```

Якщо є міграції: спершу дамп (`systemctl start transfin-pgdump.service` + відкладена копія), репетиція на відновленій копії з `upgrade → downgrade → upgrade`, лише потім справжня база, з `PGOPTIONS="-c lock_timeout=5s -c statement_timeout=60s"`. Після: `systemctl is-active`, `https://transfin.uk` → 200, `/api/...` без входу → 401, `journalctl -u transfin.service` без помилок. Для recorder — ще й `journalctl -u transfin-etoll-recorder | grep -c password=` → 0.

**Downgrade `e7011a5c3b19` видаляє схему `etoll` з даними — лише з окремого явного дозволу власника.**

---

## 2. Що на сервері зараз (21.09)

- **На сервері:** код `961387e` — P1–P8 і P5a (розгорнуто 21.09, §6). Сайт `active`, recorder `active`/`enabled` (не перезапускався, його код не змінювався). Ledger — `0006_audit_actor`. Нерозгорнутого немає.
- **Облікові записи сайту:** активний лише id 1 (власник). id 2 (ADMIN+LOGISTICIAN) неактивний з 17.09 ~12:26 UTC — див. §6.
- **Журнал на сервері порожній:** `trip_journal`, `trip_journal_legs`, `routes`, `counterparties` — 0 рядків (з 16.09 кожна відповідь `/api/journal` — `[]`). Дані журналу — у локальній базі Mac; перенесення — окрема задача (§7), не почата.
- **Роль `FINANCE`** для другої людини: тепер можна створити акаунт (роль є в картці персоналу); бачитиме лише «Фінанси».

---

## 3. Воркер eCherha на Mac (варіант B')

eCherha захищена від ботів (headless блокується, вхід — людина й капча), тому воркер працює на Mac власника у видимому згорнутому Chrome і пише в серверну базу через SSH-тунель.

- **Тунель:** LaunchAgent `com.transfin.db-tunnel`, ssh `127.0.0.1:15432` → сервер `127.0.0.1:5432`, користувач `transfin-tunnel`, ключ `~/.ssh/transfin_tunnel_ed25519`, окремий `~/.config/transfin/known_hosts`. У plist уже є `ServerAliveInterval=15`, `ServerAliveCountMax=3`, `ExitOnForwardFailure`, `KeepAlive`, `ThrottleInterval=60`.
- **Воркер:** LaunchAgent `com.transfin.queue-worker` → `scripts/queue_worker_launchagent/run_worker.sh` → `python3.12 -m app.queue_worker` з `~/Projects/backend`.
- **Приватні файли (0600, поза git):** `~/.config/transfin/worker.env` (`DATABASE_URL` без пароля + перемикачі: `QUEUE_DOCUMENT_REMINDERS_ENABLED=false`, `QUEUE_TELEGRAM_ENABLED=false`, `QUEUE_DRIVER_DETAILS_ENABLED` не задано = вимкнено), `~/.config/transfin/pgpass` (пароль `transfin_worker`), `~/.config/transfin/known_hosts`. Шаблони plist і `install.sh`/`uninstall.sh` — `scripts/queue_worker_launchagent/` у репозиторії.
- **Профілі Chrome:** `~/Library/Application Support/TransFin/echerha_profiles/{1,2,3}` (~186 МБ). Сесії живуть тут і на інший комп'ютер не переносяться. Кеш сторінок: `~/Library/Caches/TransFin/echerha_profiles` (803 МБ; видаляти лише при зупиненому воркері).
- **Облікові записи eCherha** (id на сервері й Mac однакові):

  | id | Логін | Компанія |
  |---|---|---|
  | 1 | trans1321@icloud.com | ТОВ «ВОТРАНС» |
  | 2 | trans1321@gmail.com | ПП «Транс Левел» |
  | 3 | mv17111988mv@gmail.com | ПП «Транс Левел» |

- **Керування:**
  ```bash
  launchctl print gui/$(id -u)/com.transfin.queue-worker
  launchctl kickstart -k gui/$(id -u)/com.transfin.queue-worker
  launchctl bootout gui/$(id -u)/com.transfin.queue-worker
  launchctl disable gui/$(id -u)/com.transfin.queue-worker
  launchctl enable gui/$(id -u)/com.transfin.queue-worker
  launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.transfin.queue-worker.plist
  ```
  Те саме для `com.transfin.db-tunnel`. Зупиняти воркер, потім тунель; запускати навпаки.
- **Логи:** `~/Library/Logs/TransFin/queue_worker.err.log` (росте ~10 МБ/добу, ротації немає), `~/Library/Logs/TransFin/db_tunnel.err.log`. Корисні рядки: `Queue worker starting: … driver_details=False`; `Active Queue polling succeeded: checked=… saved=… errors=…`; `Active Queue cycle took …s` (норма ~140–155 с на 3 акаунти); `echerha account result account_id=… status=…`; `… database unavailable (…)` — короткий обрив тунелю.
- **Відомо й прийнято:** кожен перезапуск воркера — три короткі спалахи вікон (по Chrome на акаунт).
- **На сервері:** хто тримає lock — `pg_locks` + `pg_stat_activity` (`transfin-queue-worker@<Mac>:<pid>`); стан акаунтів — `company_external_accounts` (`last_read_status`, `last_success_at`, `login_request_status`).
- **Живлення:** від мережі `sleep 0`; **від батареї `sleep 5`** — Mac засинає, тунель рветься, воркер виходить і перезапускається (три нові вікна Chrome). Воркер надійний лише під живленням. 21.09 зарядка кілька годин не йшла, хоча власник підключив кабель (`pmset` показував Battery Power) — перевірити кабель/адаптер.

### Mac: диск і пам'ять (20–21.09)

- 72 «клони підпису коду» Chrome (`/private/var/folders/…/X/com.google.Chrome.code_sign_clone`) видалено скриптом `~/transfin_chrome_cleanup.sh`: звільнили **лише ~2 ГіБ** (`du` показував 100 — спільні блоки APFS). Автоприбирання (`--install-agent`) — гігієна, не потреба.
- Решту місця займають системні знімки `com.apple.os.update-*` та захищені теки — не чіпати; картина в «Сховищі». Великі теки, які власник вирішує сам: `~/Library/Application Support/Claude` (12 ГіБ), `Google/Chrome/OptGuideOnDeviceModel` (4 ГіБ), сміття тестів `/T/pytest-of-andreverum` (2 ГіБ).
- Власник видалив `~/Library/Caches/ms-playwright` — **воркеру це не шкодить**: усі точки запуску браузера використовують `channel="chrome"` (системний Chrome). Браузер Playwright знадобився б лише для запуску без цього параметра — тоді `playwright install chromium`.
- 21.09 Mac був у **важкому свопі** (7,5 з 9,2 ГБ, 5,36 млн вивантажень): повні прогони тестів тривали 40+ хв замість 6. Причина — Chrome (три вікна воркера + браузер власника), Claude, Telegram.

---

## 4. Тимчасові заходи

- **`REVOKE UPDATE ON public.crossing_log FROM transfin_worker`** (19.09, 13:24 Київ). Зупиняє безлімітні повтори читання деталей архіву eCherha. Архів стоїть; Active Queue, навантаження, алерти, вхід — працюють. **Повернення** — лише разом із лімітом спроб і підходом C1 для сторінок архіву: `GRANT UPDATE ON public.crossing_log TO transfin_worker;`. На 20.09 у черзі 11 записів без деталей — ліміт має бути в тому ж коміті, що й `GRANT`.
- **Копії `transfin.env` на сервері** (усі `root:transfin 640`, містять діючі на той момент секрети), **видалити після 27.09.2026**:

  | Копія | Навіщо |
  |---|---|
  | `transfin.env.bak-tekson-20260920T144808Z` | перед першим записом доступу Tekson |
  | `transfin.env.bak-etoll-20260920T174436Z` | перед записом `ETOLL_DATABASE_URL` |
  | `transfin.env.bak-tekson-20260920T181107Z` | перед записом перевипущеного доступу Tekson |

  Команда: `rm -f /etc/transfin/transfin.env.bak-*`.

---

## 5. Інциденти

- **19.09 — паролі IKK e-TOLL** обох компаній лежали відкритим текстом у старому `etoll.py` (iCloud) і потрапили у вивід інструментів сесії: мій фільтр маскування не впізнав ключі в лапках. **Паролі змінити** (чекає на власника). Урок: кожен фільтр маскування спершу перевіряти на фейковому рядку.
- **20.09 — доступ Tekson у журналі сервера.** `httpx` за 17 с записав 7 рядків із повним URL (логін і пароль) у journald; ті самі значення — у вивід сесії. Сервіс зупинено, доступ **перевипущено власником 20.09**, у коді — інваріант 12 (приглушення сторонніх логерів + фільтр + тест). Журнал **не чистили** (зачистка стерла б усю історію, значення все одно змінене). Після запуску на виправленому коді: `password=` у журналі — 0.
- **20.09 — код доступу Tekson у чаті.** «Очищений» URL показував логін у шляху; виправлено (`redact_url` маскує середину шляху). Пароль не засвітився; перевипуск не знадобився.
- **21.09 — тести ходили в ПриватБанк.** Тест-обхід маршрутів із P1 на кожному повному прогоні (≈10 разів за 20–21.09) робив справжній запит на виписки з `~/.privatbank_credentials.json`. Лише читання, нічого не записано. Виправлено в `91fd334`; глобальна заборона мережі в тестах — у беклозі.
- **21.09 — права файла ключів ПриватБанку:** `~/.privatbank_credentials.json` має **`644`** (читає група `staff`, тобто всі користувачі Mac, і всі інші). Виправлення — `chmod 600 ~/.privatbank_credentials.json` (чекає на власника).

---

## 6. Журнал перевірок

- **Т4 (20.09):** нічне вікно 19.09 16:45 → 20.09 16:39 — 0 перезапусків, 0 watchdog, 0 дублікатів, цикл Active Queue ~143 с; сесії пережили ніч; AC4497EO відкритий обґрунтовано (ще в черзі). Відхилення від «чистої» ночі: ручні читання архіву 19.09 вдень і `REVOKE` (§4).
- **R1, жива перевірка (20.09, прийнято із зауваженням):** акаунти 1 і 2 — `success` з першої спроби (10:53:39, 10:55:51), акаунт 3 — з четвертої (10:56:06, 10:58:14, 10:58:25 → 10:59:58, ~4 хв). Вікно не відкривалось.
- **e-TOLL СТОП 2 (20.09):** дамп + відкладена копія; `GRANT CREATE ON DATABASE transfin_backend TO etoll_owner`, `CREATE EXTENSION btree_gist` (1.8), `GRANT CONNECT … TO etoll_recorder`; репетиція на відновленій копії (`upgrade → downgrade → upgrade`); код `9705ffc → 5a296e0`, міграція `e7011a5c3b19` застосована; пароль `etoll_recorder` згенеровано на сервері й записано в базу (SCRAM) і `transfin.env` без друку. Recorder запущено 18:12 UTC на `8530c53`. 10 хв: 112 опитувань, 0 невдалих, 17 пристроїв, 154 мс, 41 МБ пам'яті; 8 пристроїв із застиглим годинником у карантині (`too_old`, не множаться); у русі 0 тягачів. 17 прив'язок пристрій↔тягач **підтверджено власником 20.09** (`confirmed_by = 1`).
- **Мовчазні пристрої Lontex (20.09):** 8 із 17 не звітують 2–8 діб, 7 із них — на тягачах у черзі на кордон. Версія власника: водії знеструмлюють пристрій в Україні (RMPD/SENT потрібні лише в Польщі). Довідка для Lontex: `~/TransFin-private/lontex_silent_devices_20260920.md` (справжні IMEI — поза git).

---

- **P1–P8 + P5a, розгортання (21.09, 08:09–08:30 UTC):** дамп обох баз + відкладені копії `pre-p1p8-*_20260921T080917Z`; код `62bbf44 → 961387e` (12 комітів, без змін залежностей і коду recorder); репетиція `0006` на відновленій копії `transfin` (`upgrade → downgrade → upgrade`, хеш тіла тригера `5a7fe6… → a58f48… → 5a7fe6…`, перевірка: з актором — `rehearsal:p1p8`, без — `db_trigger`; копію видалено); `0006` на справжній `transfin` з `lock_timeout=5s`; `ALLOWED_HOSTS` і перезапуск сайту — **виконав власник** (автоперевірка Claude Code блокувала ці команди); перевірки: `/` 200, `/api/*` без входу 401, чужий Host 400, сесії пережили перезапуск; власник після перезавантаження сторінки відкрив усі розділи — 0×403, 0×5xx, 0 помилок застосунку; `/api/ledger/currencies` 200. Воркер: перший цикл Active Queue після перезапуску — `checked=3 saved=14 errors=0`. Recorder не чіпали: 0 перезапусків, `password=` — 0. Спостереження 14 хв (до 08:30 UTC): 432×200, 2×401 (перевірки без входу), 0×403, 0×5xx, 0 перезапусків сайту й recorder; два цикли Active Queue поспіль успішні, `last_success_at` усіх трьох акаунтів оновлюється.
- **Обліковий запис id 2 (перевірка 21.09, лише читання):** створений власником 17.09 12:11 UTC у Safari; з 12:12 по ~12:26 ним користувався **інший комп'ютер** (Mac, Chrome 152 — у власника того часу паралельно Chrome 153): 3 невдалі входи, вхід, перегляд усіх розділів, один запис — «запросити вхід» eCherha (таблицю акаунтів eCherha потім очищено й створено наново). Після деактивації — лише 401: вкладка опитувала сервер до 19.09 (інша мережа), останнє — невдалий вхід 19.09 21:14 Київ. Імовірно — сам працівник; nginx не записує користувача, тож це висновок, не доказ.

## 7. Чекає на власника

- Змінити паролі IKK e-TOLL обох компаній (§5).
- `chmod 600 ~/.privatbank_credentials.json` (§5).
- Надіслати довідку Lontex (§6) і з'ясувати: напруга живлення, час останнього пакета e-TOLL і стани SENT через API.
- Видалити копії `transfin.env` після 27.09 (§4).
- Перевірити зарядку Mac (§3).
- Перенесення даних журналу (рейси, маршрути, контрагенти) з Mac на сервер — окрема задача з планом (рішення власника 21.09: не зараз).

## 8. Інфраструктура й гігієна

- fail2ban на SSH (боти пробують входи; пароль уже вимкнено).
- **Бекапи поза сервером** — немає; Time Machine на Mac — немає.
- Стара копія репозиторію `~/Desktop/backend.MOVED-20260918` (в iCloud, у `backups/` — персональні дані): видалити лише з дозволу власника.
- **~95 копій баз із персональними даними в iCloud** (`~/Desktop`) і відкриті секрети на Mac: `~/.transfin_config.json`, паролі в `*.disabled`, `.env` в iCloud, `TELEGRAM_BOT_TOKEN` у `~/.zshrc`. Рішення власника; нікуди не копіювати й не друкувати. Права на ці файли — перевірити (як для ключів ПриватБанку).
- Скрипти на Mac поза git: `~/transfin_tekson_setup.sh` (запис доступу Tekson: значення лише через stdin, атомарна заміна файла, решта рядків звіряється `cmp`, копія лишається); `~/transfin_chrome_cleanup.sh` (прибирання клонів Chrome: без аргументів — показ, `--apply` — видалення лише `code_sign_clone.*` старших за 60 хв і не відкритих, `--install-agent` — щоденно о 05:30 через LaunchAgent `com.transfin.chrome-clone-cleanup`, лог `~/Library/Logs/TransFin/chrome_clone_cleanup.log`).
- Старі десктопні інструменти (`transfin.py`, `zvirka.py` і їхні копії) перейменовано в `.disabled` 18–19.09, щоб не читали eCherha паралельно з воркером.
