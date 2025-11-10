# Отчёт по симуляциям роевого взаимодействия дронов

## &#x20;«чистая математика» (авторские JAX-среды GCBF+)

**Цель.** Зафиксировать воспроизводимый протокол запуска и оценки GCBF+ в авторских средах без физического движка  и собрать сравнение с базовыми методами.

---

## 1. Репозитории и артефакты

- Сайт проекта: [https://mit-realm.github.io/gcbfplus/](https://mit-realm.github.io/gcbfplus/)
- Код: [https://github.com/MIT-REALM/gcbfplus/](https://github.com/MIT-REALM/gcbfplus/)
- Предобученные модели: `pretrained/` (в репозитории)
- Конфигурация гиперпараметров по умолчанию: `settings.yaml`
- Скрипты: `train.py`, `test.py`
- Логи и видео: `./logs/<env>/<algo>/seed<seed>_<timestamp>/` и подпапка `videos/`

---

## 2. Среды моделирования (2D/3D)

**2D:** `SingleIntegrator`, `DoubleIntegrator`, `DubinsCar`\
**3D:** `LinearDrone`, `CrazyFlie`

Кратко:

- *SingleIntegrator*: состояние — позиция, управление — скорость. Без инерции.
- *DoubleIntegrator*: добавлена скорость в состояние; управление — ускорение.
- *DubinsCar*: неголономная «машинка»: \(\dot p = v[\cos\theta,\,\sin\theta],\ \dot\theta=\omega\).
- *LinearDrone*: линеаризованный квадрокоптер вокруг зависания, \(\dot x = A x + B u\).
- *CrazyFlie*: 6DoF нелинейная модель с тягой/моментами.

---

## 3. Алгоритмы для сравнения

- `gcbf+` (основной метод)
- `gcbf` (предыдущая версия)
- `centralized_cbf` (централизованный CBF-QP)
- `dec_share_cbf` (децентрализованный CBF-QP)

Выбор алгоритма: флаг `--algo`.

---

## 4. Установка окружения

1. **Conda**

```bash
conda create -n gcbfplus python=3.10 -y
conda activate gcbfplus
```

2. **JAX** — установить по инструкции под вашу CUDA/CPU.
3. Зависимости проекта:

```bash
pip install -r requirements.txt
pip install -e .
```

> Примечание: результаты в статье получены на `jax==0.4.23` (GPU). Для CPU можно использовать флаг `--cpu` в `test.py`.

---

## 5. Базовые команды

### Обучение (пример)

```bash
python train.py \
  --algo gcbf+ --env DoubleIntegrator \
  -n 8 --area-size 4 \
  --loss-action-coef 1e-4 \
  --n-env-train 16 \
  --lr-actor 1e-5 --lr-cbf 1e-5 \
  --horizon 32 --steps 1000
```

Ключи: `-n` число агентов; `--obs` препятствия; `--n-rays` лучей лидарa; `--n-env-train/test` число параллельных сред и т.д.

### Тестирование (оценка + видео)

```bash
python test.py --path <path-to-log> --epi 5 --area-size 4 -n 16 --obs 0
```

Метрики печатаются в консоль и сохраняются в лог: **safety rate**, **goal reaching rate**, **success rate**; видео — в `<path>/videos/`.

### Номинальный контроллер (сравнение)

```bash
python test.py --env SingleIntegrator -n 16 --u-ref --epi 1 --area-size 4 --obs 0
```

### CBF-QP (децентрализованный)

```bash
python test.py --env SingleIntegrator -n 16 \
  --algo dec_share_cbf --epi 1 --area-size 4 --obs 0 --alpha 1
```

Полезные флаги: `--nojit-rollout` (масштабные запуски), `--max-step`, `--dpi`, `--no-video`, `--seed`.

---

## 6. Определения метрик

- **Safety rate** — доля эпизодов без коллизий.
- **Goal reaching rate** — доля эпизодов, где все агенты достигли целей.
- **Success rate** — доля эпизодов, где одновременно соблюдена безопасность и достигнуты цели.

---

## 7. Конфигурация вычислительного окружения

- Версия Python: 3.10
- JAX: *рекомендуется* 0.4.23 (GPU), но допускается CPU (`--cpu`).

Советы по производительности: уменьшайте `--dpi` (видео), отключайте видео (`--no-video`), повышайте `--area-size` и `--max-step` только при необходимости, используйте «лёгкие» среды для массовых тестов.

---

## 8. Ограничения 

- Не моделируются физические контакты и низкоуровневая аэродинамика; алгоритм сам обеспечивает коллизие-обход.
- Сенсорика сведена к «идеальной» или к синтетическому лидару.

---
