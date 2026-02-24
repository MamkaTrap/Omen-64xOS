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

