# Передача tsan-improve, 28 Oct 2026 (переезд на другой аккаунт)

Перевод файла tsan-improve-handoff-2026-10-28.md; при расхождении главный английский.

Передачи других сессий того же дня: /home/alexey/atc26-handoff/tsan-exp-handoff-2026-10-28.md и
/home/alexey/atc26-handoff/runtime/tsan-rt-handoff-2026-10-28.md.

## Роли
- **tsan-improve** (tmux `2-tsan-improve`) — координация замеров версий PR, тексты PR, остатки ATC'26.
- **tsan-exp** (tmux `2-tsan-exp`) — все сборки, замеры, обработка; цепочки идут сами на focs, сохранность гонок — на
  a-p13. Протокол: `/extra/alexey/upr-2026-10-20/REGISTERED.txt`, результаты: `/extra/alexey/upr-2026-10-20/READOUT-*.txt`.
- **tsan-rt** (tmux `2-tsan-rt`) — оптимизации рантайма для второй статьи, apollo. Не наша работа.

## Файл результатов
`/home/alexey/atc26-handoff/upstream-pr-results.md` → https://github.com/apaznikov/atc26-handoff/blob/main/upstream-pr-results.md
Всё в процентах к стоку; правило вердиктов в легенде (95 % интервал против A/A того же замера); 🟰 в столбце
«DE + runtime fix» — ещё и «нет отчётов о гонках за 5 прогонов»; однопоточных программ и запусков нет.

## Состояние замеров (28 Oct ~15:00)
Сделано: DE + правка на SQLite/memcached/MySQL (всё «=», кроме тестов SQLite); сохранность гонок — ни одной потери, в том
числе на FFmpeg с декодированием HEVC; подсчёт гонок (🟰 у zstd, xz, brotli, LevelDB, OpenSSL, 7-Zip, FFmpeg, Redis -O0);
ядра на 8 потоках — разрешился только NPB LU W (DE +103 %), остальные 🟡/= из-за собственного разброса 8–20 %; SQLite -O0
по тестам 🟢 во всех вариантах; Redis LRANGE −2…3 % с DE; Chromium 156 не разрешён, один тест dom ~−3 %.

В очереди (до ~31 Oct–1 Nov): мелкие ячейки приложений и RocksDB; NPB LU A (переделка); I12 — DuckDB, сборки Chromium
«только DE»/«только EA», замер Chromium на 5 повторах, DE + правка для строк с гонками (pigz, DuckDB, hotspot, ocean_ncp,
streamcluster, BT), профиль Redis LRANGE. AMD — только после Intel.

При возобновлении: прочитать новые READOUT и `/extra/alexey/upr-2026-10-20/STATUS-LOG.md` (журнал tsan-exp), внести в
таблицу, пересчитав вердикты по правилу, опубликовать, доложить.

## PR в LLVM
Правка рантайма входит в #169897 (решено 26 Oct). Локальная ветка `upstream-next/dom-final-rt` (4a134ba07eae): NFC →
правка рантайма (4139d1043f04, аудит без блокеров) → DE + тест. Тесты: check-tsan 383/0, lit прохода 26/0, отрицательный
контроль ловит дыру. Не выложено. Черновик описания: `/extra/alexey/upstream-next/pr-texts/169897-description.md`.
Push — только с явного «да» на каждый.

## ATC'26
Корректура TAPS от 26 Oct содержит все 48 правок и запятую — нужно нажать Approve. Открыто у пользователя: форма
докладчика, ответ председателям, шеферд, A7, CHECK PAPER DETAILS, письмо для визы, гостиница G-ACM1, Concur/Trip.Biz,
доклад и постер. Конференция 15–18 Nov, Гонконг.

## Решения за пользователем
1. AMD — когда и нужно ли.
2. Неразрешённые ядра (lavaMD, srad, particlefilter, water_spatial) — давать в тексте PR как 🟡 с устойчивыми оценками или убрать.
3. Когда выкладывать `upstream-next/dom-final-rt` в #169897.
