# Omen 64xOS

> Хобби-проект 64-битной ОС с фокусом на системное программирование: boot-chain, ядро, scheduler, memory manager, syscall ABI и собственная FS.

## Omen 64xOS v0.1 — технический срез текущего состояния

---

##  Текущий статус

Фаза: стабилизация файловой системы (production hardening).

---

##  Многопоточность и процессы

- Preemptive SMP scheduler
- Полноценная модель процессов и потоков
- Состояния: `Ready / Running / Blocked / Terminated`
- Wait / wake / sleep / timeout
- Контрактные переходы состояний
- Cleanup / reaper механизмы
- Поддержка preemption и межпроцессорных прерываний
- Инварианты планировщика и контроль корректности переходов

Планировщик стабилен и используется как основа для всех подсистем.

---

##  Syscall Interface

- Переход из user-mode в kernel-mode
- Контролируемый syscall-диспетчер
- Изоляция адресных пространств
- Базовая модель user/kernel boundary

Syscall-слой формирует основу для будущей user-mode среды.

---

##  Подсистема памяти

- PMM / VMM
- Paging для ядра и user-space
- Изолированные address spaces
- Copy-on-Write (COW)
- NX защита
- Guard pages
- Page fault handling
- Section cache

Модель памяти интегрирована с планировщиком и VM-слоем.

---

##  Файловая система — OMFS

Собственная файловая система с NTFS-подобной on-disk моделью:

- VBR + OMFT + bitmap + logfile
- Resident / non-resident атрибуты
- Runlist / mapping pairs
- Directory index
- Redo WAL
- Recovery после сбоев

Текущий фокус — стабилизация write path, recovery и lock-иерархий.

---

##  Производительность (текущие тесты)

Тест: создание 10 файлов по 1 МБ  
- **Скорость записи:** 14.56 файлов/сек

Тест: удаление файлов  
- **Скорость удаления:** 64.94 файла/сек

Показатели будут обновляться по мере оптимизации write-path и кэша.

---

##  Boot & Platform

- Stage1 → Stage2 → KernelMain
- Long mode
- APIC / IOAPIC
- Базовая инициализация платформы

---

##  Ближайшие задачи

- Финализация WAL-инвариантов
- Дополнительные стресс-тесты OMFS
- Профилирование write-path
- Расширение syscall-интерфейса

---

##  Что активно дорабатывается

- Снижение contention и задержек в горячих участках scheduler/FS.
- Усиление длительной стабильности на длинных stress-run профилях.
- Расширение user-mode совместимости и ABI-поверхности.
- Дальнейшее усиление recovery/acceptance-критериев перед следующим релизным этапом.

---

###  Технические результаты
- Throughput: для текущего цикла `omfs__fsyncgate` throughput-gate не применялся (`Aggregates.Throughput.Samples=0`); фокус цикла — tail-latency и durability.
- Latency (p95/p99): по принятому `10x60` `write=8/8 ms`, `fsync=1/1 ms`; дополнительные op-latency: `lookup=7/7 ms`, `create=18/18 ms`.
- Stability/Recovery: strict chain `1x60 -> 3x60 -> 1x180 -> regress 1x60 -> 10x60` пройден, `AttemptCount=1` во всех 10 раундах `10x60`, маркеры отказов/паники/timeout в принятом артефакте отсутствуют (`MountFail=0`, `NoPass=0`, `TimeoutWithoutStrictPass=0`, `FailMarkers=0`).


---

### Обновление v0.2

1. Загрузчик и запуск на железе
- UEFI loader обновлен до `v3`;
- загрузка `kernel.bin` и `omenxos.img` через UEFI;
- поддержка GOP framebuffer handoff;
- улучшена совместимость загрузки по UEFI-путям (`BOOTX64.EFI`, `bootmgfw.efi`);
- доработан стартовый вывод (меньше диагностического шума на экране).

2. Встроенный UEFI-инсталлятор
- установка запускается прямо из загрузчика (`I` в первые секунды);
- выбор целевого диска в интерфейсе установщика;
- подтверждение опасных операций (erase/install);
- режим установки на диск с проверками прогресса и верификацией записи;
- после успешной установки полноценно запускается и работает с диска.

3. Подготовка установочной флешки
- добавлен полноценный сценарий подготовки UEFI USB одним скриптом;

4. Ядро и системный API
- расширен syscall-слой (`event`, `file`, `handle`, `input`, `job`, `memory`, `perf`, `process`, `smp`, `sync`, `thread`, `tls`, `version`);
- усилены подсистемы планировщика и синхронизации (включая `APC`/`DPC`/timer paths и расширенный тестовый контур);
- добавлены/расширены системные проверки состояния ядра и объектов (`locks/irq/threads/scheduler/object audit`).

5. Userspace-трек (крупный прогресс относительно v0.1)
- реализованы модули userspace-валидации и сопровождения:
  - `userspace_compat_matrix`
  - `userspace_package_metadata`
  - `userspace_promotion`
  - `userspace_upgrade_recovery`
  - `userspace_timeline`
  - `userspace_incident_replay`
  - `userspace_observability`
  - `userspace_crash_dump`
  - `userspace_failure_triage`;
- реализован процессный/сервисный контур:
  - `service_control`
  - `session_orchestrator`
  - `session_environment`
  - `session_shell_policy`
  - `session_cleanup_policy`
  - `job`;
- добавлены gate-сценарии userspace от `u0` до `u9`, включая failover, security, API/refdll, session, automation, matrix/unsupported, promotion/recovery, replay/timeline/triage, crashdump/observability и soak/storm.

6. Файловая система OMFS (большой объем доработок)
- расширены функциональные сценарии:
  - `ADS`
  - `snapshot`
  - `clone`
  - `backup/restore`
  - `reparse`
  - `share`
  - `security`
  - `HA` (`test/rebuild/scrub`)
  - `data checksums`
  - `DR`;
- расширены стресс/производительные и fault-сценарии:
  - `omfs_writebench`
  - `omfs_fsyncgate`
  - `omfs_mtstress`
  - `omfs_stress`
  - `omfs_txfault`
  - `omfs_fuzz`
  - `omfs_cigate`;
- добавлен/усилен набор gate-скриптов для надежности и регрессий:
  - atomicity
  - replay idempotency
  - metadata checksums
  - orphan cleanup
  - checkpoint predictability
  - feature flags
  - versioning
  - DR/RAS.

7. Shell и интерфейс
- расширен набор пользовательских команд;
- добавлена команда установки из shell: `install`;
- исправлены визуальные артефакты framebuffer-консоли (полосы/подчеркивания под цветным текстом);
- улучшено поведение курсора при печати.

8. Валидация и автоматизация
- добавлен большой набор CI/gate-скриптов для OMFS и userspace;
- появились отдельные сценарии длительных прогонов, fault-injection и regression-проверок;
- усилен контроль стабильности (не только “запустилось”, но и “прошло серию нагрузок”).
---

### Обновление v0.3 — текущий срез архитектуры

1. Архитектурный переход
- приоритет смещен на фундамент: `scheduler`, потоки, процессы, память;
- файловый путь переводится на событийную и шардированную модель;
- слабые синхронные hot-path контуры выводятся из нормального пути.

2. Ядро и многопоточность
- сохранена и усиливается preemptive `SMP`-модель;
- усилена изоляция `process/thread/memory`;
- `FileIO/OMFS` выводятся из `kernel stack`;
- база готовится под честную работу на `1 CPU` и `SMP`.

3. FileIO / OMFS
- write path переведен на более асинхронную схему;
- вводится `sharded dirty-extent` модель;
- `writeback` разделен на `mapping / transport / completion`;
- `partial RMW` переводится в двухфазный async transport;
- `metadata / prefetch / byte-IO / I30 scratch` заранее подготавливаются без hot-path heap growth.

4. Userspace / compatibility
- закреплена Omen-native модель внутренних сервисов;
- оформляется внешний `DLL/API` compatibility layer;
- закладывается база под полноценный `userspace`, `services`, `subsystem` и `UI`.

5. Registry / subsystem / UI
- подготовлен совместимый `registry namespace`;
- введена более явная схема DLL/API-совместимости;
- `subsystem` и `UI` выведены в отдельные архитектурные треки;
- система готовится к единому `userspace/API` слою для приложений, сервисов и GUI.

6. Что убрано и что уходит
- old-style direct/raw fallback выводится из нормального write path;
- снижается зависимость от широких синхронных участков;
- weak scratch/runtime growth на горячем пути убирается;
- старые слабые контуры больше не считаются целевой архитектурой.

7. Метрики
- текущий рабочий `SMP=4` baseline:
  - `avg_mibps_dec = 1016.35 MiB/s`
  - `avg_write_ms = 16 ms`
  - `avg_delete_ms = 556 ms`
- отдельная публикация `delete`:
  - `delete_publish = 7 ms`
- максимум удержания write-резиденции:
  - `claimext_write_bytes_max: 8 MiB -> 1 MiB`
- лучший зафиксированный throughput reference:
  - `1090.90 MiB/s`
- лучший throughput-only snapshot:
  - `1137.25 MiB/s`

8. Текущее состояние
- `small/medium write path` стал заметно сильнее;
- `delete path` ускорен, но еще не доведен до целевого low-latency уровня;
- большой файловый путь пока остается слабым местом;

9. Что активно добивается
- дальнейшее снижение `delete` latency;
- устранение large-file serialization;

### Обновление v0.3.1

1. Загрузка и bring-up
- есть `UEFI` loader с загрузкой `kernel.bin` и образа диска через `Block I/O`;
- есть handoff `GOP framebuffer`, `ACPI` таблиц и геометрии boot image в ядро;
- есть встроенный установщик прямо в `UEFI`-загрузчике;
- сохранены asm-загрузчики (`loader64.asm`, `loader64_hw.asm`);
- весь ранний handoff оформлен через `shared/boot_info.h`.

2. Scheduler / runtime ядра
- есть preemptive `SMP` scheduler с process/thread object model;
- есть `wait/wake/sleep/timeout`, `APC`, `DPC`, timer и cleanup/reap paths;
- есть per-CPU runtime и published current-thread ownership;
- есть собственный thread-object allocation path с thread-pool slots и kernel stack guard pages;
- есть object manager и handle table как общая runtime-база.

3. Процессы, потоки и session control plane
- есть `proc_manager`, `job`, `service_control`, `session_orchestrator`, `session_environment`, `session_shell_policy`, `session_cleanup_policy`;
- bootstrap разделен на runtime/service/session события, а не свален в один стартовый поток;
- у `session_orchestrator` уже есть явные состояния `WaitingBootstrap -> SessionReady -> LogonReady -> ShellAttached -> InteractiveReady -> Running`;
- startup shell публикуется самой shell-thread, а не только внешним supervisor path;
- cleanup и environment teardown уже вынесены в отдельный control-plane контракт.

4. Syscall ABI и userspace boundary
- есть syscall entry через `MSR` и центральный dispatcher;
- есть syscall-поверхность: `event`, `file`, `handle`, `input`, `job`, `memory`, `perf`, `process`, `smp`, `sync`, `thread`, `tls`, `version`;
- есть `PEB/TEB`, `TLS`, `QueryPerf`, thread/process identity paths;
- есть compat export/syscall map для `ntdll.dll` и `kernel32.dll`;
- в `userspace` уже лежат smoke/bench-программы, которые ходят в этот ABI напрямую.

5. Память и загрузка образов
- есть paging, address spaces, `VAD` tree и `user_copy`;
- `VAD` уже несет `commit/reserve/file/image/COW/guard` семантику;
- есть section objects и section cache;
- у section cache уже есть отдельные demote/reclaim background workers;
- есть `PE` loader и `ELF` loader в kernel-side image loading path;
- `NX`, guard pages и page-fault-backed section mapping уже встроены в VM-путь.

6. FileIO surface
- есть `VolumeManager` с drive-letter routing;
- есть kernel file API для `open/create/read/write/append/flush/set-size/lock-range`;
- blocking file path и async file path уже живут в одном стеке;
- async file path уже имеет completion events, completion ports, `IoTicket`, `IoToken`, `IoFence` и `APC` completion routines;
- blocking wrapper path уже умеет дожидаться и освобождать `IoTicket`, а не существует отдельной полностью чужой реализацией.

7. OMFS как текущая файловая архитектура
- в дереве уже есть `volume`, `logfile`, `tx`, `OMFT`, `bitmap`, `attr`, `runlist`, `create/open/rename/delete`, `I30`, `mount/recovery`, `checkpoint`, `cluster cache`, `security metadata`, `reparse/share/ADS/clone/snapshot/backup` paths;
- namespace mutation оформлен отдельным слоем, а не локальной логикой внутри каждого file operation;
- metadata tree publication тоже оформлен отдельным набором generation/publish/read/retire доменов;
- mount path заранее поднимает dedicated scratch planes для metadata, `I30`, prefetch, byte-IO и prepared physical async block batches.

8. Уникальные runtime-механизмы OMFS, которые уже есть
- allocator уже разбит на planes: `data`, `metadata`, `namespace`, `tx-lifecycle`, `mount-state`, `rebalance`, `mount-bootstrap`;
- есть `OmfsAllocatorCursorStateService` с общим атомарным shared hint cursor, priming от materialized records и forward-only publication;
- есть adaptive runtime state с metadata-domain locks, secure-id cache, path cache и directory-lookup cache;
- path cache и dir-lookup cache уже шардированы и держат per-shard locks, free lists и `LRU` state;
- volume runtime уже различает bootstrap direct lane и дальнейшие active I/O lanes;
- часть hot-path progress/cursor state уже публикуется атомарно через shared hints и sequences, а не только через broad global lock.

9. OMFS writeback / durability / background ownership
- есть `OmfsDirtyExtentEngine` с многополосной моделью (`lanes`), per-lane queue/active/continuation state, wake/completion events и progress sequence;
- есть authority model для dirty extents по `cacheEntryId / fileRef / mappingEpoch / fileRefBucket`;
- есть `OmfsSyncAsyncWaveService` с wave tickets, `targetCommitSeq`, `targetLsn`, policy bits и операциями `join/publish/requeue/complete`;
- `tx` уже разделяет commit planes (`data / metadata / control`) и ведет plane-specific log/apply ledgers;
- есть `OmfsCheckpointBackgroundService` с backend hooks для snapshot publication, deferred record release, allocator free-cluster replay, committed-root publication и create durable-apply queues;
- есть `durability progress` сервис и отдельные checkpoint control/sequencing/scheduling/runtime слои.

10. Security, registry и service data
- есть `token`, `ACL`, integrity и default `DACL` model;
- внутри `OMFS` уже есть secure catalog, secure cache, security-id transition и descriptor apply paths;
- есть `registry_core`;
- у registry уже есть memory adapter и `OMFS` adapter;
- у registry keys уже есть access-mask и security-descriptor logic внутри кода;
- service/session/userspace control data уже вынесены в отдельные kernel модули, а не живут только в shell-командах.

11. Интерактивная поверхность и драйверы
- текущая интерактивная поверхность системы — kernel console shell;
- shell уже содержит большой набор operational/gate команд для `OMFS`, scheduler, памяти, registry, security, block/I/O и userspace tracks;
- в дереве есть драйверы `APIC`, `timer/clock/PIT`, `serial`, `VGA text`, `keyboard`, `ATA PIO`, `boot ramdisk`, `mirror block device`;
- userspace-трек в текущем виде уже включает `ref_libs`, bench/smoke-программы и kernel-side support-модули для compat matrix, package metadata, promotion, recovery, replay, timeline, crash dump и observability.
