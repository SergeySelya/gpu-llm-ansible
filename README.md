# GPU-сервер под LLM: Ansible

Плейбук готовит чистый Ubuntu-сервер с NVIDIA GPU и поднимает на нём LLM с OpenAI-совместимым API:

1. Драйвер NVIDIA из архива Ubuntu (серверная ветка R580), перезагрузка, только если драйвер поставился.
2. NVIDIA Container Toolkit и Docker Engine с compose-плагином. Runtime `nvidia` прописан в `daemon.json`.
3. vLLM v0.34.0 с моделью из Hugging Face. По умолчанию это `Qwen/Qwen2.5-1.5B-Instruct`: без токена HF, влезает в 8 ГБ видеопамяти.
4. Перед vLLM стоит nginx: TLS, allowlist по сетям, rate limit. Сам vLLM наружу не опубликован.
5. dcgm-exporter отдаёт метрики GPU для Prometheus.
6. В конце плейбук проверяет всю цепочку: health, 401 без ключа, список моделей, настоящий запрос к модели, метрики GPU.

```
                      сервер (Ubuntu + NVIDIA driver + container toolkit)
 клиент ──HTTPS:443──► ┌─────────────────────── docker compose (/opt/llm) ───────────────────┐
  Bearer <key>         │  nginx ── TLS, allow CIDR, 10r/s ──► vllm:8000 (OpenAI API, GPU)    │
                       │                                        └─ /opt/llm/hf-cache (веса)  │
 Prometheus ──:9400──► │  dcgm-exporter (GPU)                                                │
                       └─────────────────────────────────────────────────────────────────────┘
```

## Требования

Машина, с которой запускаете (control node):
- `ansible-core` 2.16 или новее;
- коллекции из `requirements.yml`;
- SSH-доступ к серверу пользователем с sudo.

Сервер:
- Ubuntu 22.04, 24.04 или 26.04, x86_64;
- GPU NVIDIA Turing и новее (T4, RTX 20xx, A10, L4, A100, H100 и т.д.). Образ vLLM v0.34 собран на CUDA 13.0, а CUDA 13 не поддерживает Maxwell, Pascal и Volta (P100, V100);
- для модели по умолчанию хватит 8 ГБ VRAM. Под 7B/8B-модели в bf16 нужно 24 ГБ и больше;
- около 40 ГБ на диске под `/opt` и `/var/lib/docker`: образ vLLM весит порядка 10 ГБ, плюс веса модели;
- выход в интернет к `archive.ubuntu.com`, `download.docker.com`, `nvidia.github.io`, Docker Hub, `nvcr.io` и `huggingface.co`.

## Запуск

```bash
ansible-galaxy collection install -r requirements.yml -p ./collections
```

```bash
cp inventory/hosts.yml.example inventory/hosts.yml
```

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
```

Впишите в `vault.yml` ключ API (`openssl rand -hex 32`) и, если модель gated, токен HF. Потом зашифруйте файл:

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

```bash
ansible-playbook site.yml --ask-vault-pass
```

Первый прогон занимает 10–30 минут: ставится драйвер, сервер перезагружается, скачиваются образ и веса. Повторный прогон идемпотентен. Драйвер не переустанавливается, если активна нужная ветка; контейнеры пересоздаются только при изменении конфигурации.

Модель, сети и прочее настраиваются в `group_vars/all/main.yml`. Все переменные с дефолтами и комментариями лежат в `roles/*/defaults/main.yml`.

Отдельные шаги можно запускать тегами: `common`, `driver`, `toolkit`, `docker`, `llm`, `verify`. Например, только проверка уже развёрнутого стека:

```bash
ansible-playbook site.yml --tags verify --ask-vault-pass
```

## Как работает проверка

Роль `llm` заканчивается `tasks/verify.yml`. Если любой шаг не прошёл, прогон падает:

| Шаг | Что подтверждает |
|---|---|
| `nvidia-smi` после установки и проверка версии драйвера ≥ R580 | модуль ядра загружен, драйвер подходит под CUDA 13 из образа |
| compute capability каждой GPU ≥ 7.5 | карта поддерживается CUDA 13, а не упадёт в vLLM с `no kernel image` |
| `docker run --gpus all ubuntu:24.04 nvidia-smi -L` | контейнеры видят GPU через toolkit |
| `GET /health` через nginx, ждём до 15 минут | vLLM загрузил модель в GPU |
| `GET /v1/models` без ключа → 401 | API закрыт ключом |
| `GET /v1/models` с ключом | отдаётся нужная модель |
| `POST /v1/chat/completions` | модель отвечает, ответ не пустой |
| `GET :9400/metrics` содержит `DCGM_FI_DEV_GPU_UTIL` | метрики GPU собираются |

Ответ модели выводится в лог прогона. Пример того, как это выглядит:

```
TASK [llm : Show model reply]
ok: [gpu-01] => msg: 'qwen2.5-1.5b: pong (38 tokens)'
```

## Ручная проверка на сервере

```bash
nvidia-smi
```

```bash
docker run --rm --gpus all ubuntu:24.04 nvidia-smi -L
```

```bash
sudo docker compose -f /opt/llm/compose.yml ps
```

```bash
sudo docker compose -f /opt/llm/compose.yml logs -f vllm
```

## Пример запроса к API

Сертификат по умолчанию самоподписанный. Для теста подойдёт `-k`, в работе выпустите нормальный сертификат (см. `llm_tls_cert_file`) или передавайте клиенту `/opt/llm/certs/tls.crt`.

```bash
export LLM_URL=https://gpu-01.example.internal LLM_KEY=<ключ из vault>
```

```bash
curl -k $LLM_URL/v1/models -H "Authorization: Bearer $LLM_KEY"
```

```bash
curl -k $LLM_URL/v1/chat/completions \
  -H "Authorization: Bearer $LLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "model": "qwen2.5-1.5b",
        "messages": [{"role": "user", "content": "Объясни, что такое KV-cache, в двух предложениях"}],
        "max_tokens": 200,
        "temperature": 0.2
      }'
```

Ответ приходит в формате OpenAI (`choices[0].message.content`, `usage`). Для стриминга добавьте `"stream": true`: nginx отдаёт SSE без буферизации.

Через OpenAI SDK для Python:

```python
import httpx
from openai import OpenAI

client = OpenAI(
    base_url="https://gpu-01.example.internal/v1",
    api_key="<ключ из vault>",
    http_client=httpx.Client(verify="tls.crt"),  # самоподписанный сертификат с сервера
)
resp = client.chat.completions.create(
    model="qwen2.5-1.5b",
    messages=[{"role": "user", "content": "Привет! Ответь одним словом."}],
)
print(resp.choices[0].message.content)
```

## Автозапуск и восстановление

- `docker.service` включён, у всех контейнеров `restart: unless-stopped`. После перезагрузки сервера стек поднимается сам, веса берутся из `/opt/llm/hf-cache` без повторной загрузки.
- `nvidia-persistenced` держит драйвер инициализированным, поэтому первый запрос к GPU не ждёт его инициализации.
- У vLLM есть healthcheck на `/health`. `start_period` 30 минут, чтобы первая загрузка весов не считалась сбоем. Статус видно в `docker compose ps`.
- nginx перерезолвит `vllm` через DNS Docker, так что пересоздание контейнера vLLM не ломает прокси.

## Мониторинг GPU

Источников метрик два:

| Источник | Адрес | Что смотреть |
|---|---|---|
| dcgm-exporter 4.8.4 | `:9400/metrics`, по умолчанию только `127.0.0.1` | `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_FB_USED`, `DCGM_FI_DEV_GPU_TEMP`, `DCGM_FI_DEV_POWER_USAGE`, `DCGM_FI_DEV_XID_ERRORS` |
| vLLM | `https://<host>/metrics`, доступ из `llm_metrics_allowed_cidrs` | `vllm:num_requests_running`, `vllm:num_requests_waiting`, `vllm:kv_cache_usage_perc`, `vllm:time_to_first_token_seconds`, `vllm:e2e_request_latency_seconds` |

dcgm-exporter запинен на 4.8.4 намеренно. В 4.9.0 переименованы метрики и поменялись единицы (утилизация стала долей, память считается в байтах), из-за чего готовые дашборды ломаются. Обновлять его стоит вместе с дашбордами и алертами.

Чтобы Prometheus мог забирать метрики, задайте `llm_dcgm_exporter_listen_address` равным внутреннему IP сервера. Пример scrape-конфига:

```yaml
scrape_configs:
  - job_name: dcgm
    static_configs:
      - targets: ["10.0.10.21:9400"]
  - job_name: vllm
    scheme: https
    tls_config:
      insecure_skip_verify: true   # или ca_file с сертификатом сервера
    static_configs:
      - targets: ["10.0.10.21:443"]
```

Дашборды Grafana: «NVIDIA DCGM Exporter Dashboard» (ID 12239) и дашборд из `examples/observability` в репозитории vLLM.

Какие алерты завести в первую очередь:
- `DCGM_FI_DEV_XID_ERRORS > 0`: аппаратные или драйверные ошибки GPU;
- температура GPU выше 85 °C дольше 5 минут;
- `vllm:num_requests_waiting > 0` дольше 5 минут: очередь растёт, мощности не хватает;
- `vllm:kv_cache_usage_perc > 0.95`: KV-кэш забит, запросы вытесняются;
- `up{job="vllm"} == 0`.

## Безопасность API

- **Ключ.** vLLM без Bearer-ключа (`VLLM_API_KEY`) возвращает 401 на `/v1/*`. Ключ лежит в Ansible Vault, на сервере хранится в `/opt/llm/.env` с правами 0600. Задачи, где он участвует, помечены `no_log`, поэтому в лог прогона он не попадает.
- **TLS.** Трафик терминирует nginx, доступны только TLS 1.2 и 1.3. Можно подложить свой сертификат или оставить самоподписанный.
- **Минимум наружу.** Порт 8000 vLLM не публикуется. Снаружи доступны только `/v1/*`, `/health` и `/metrics` (последний только из `llm_metrics_allowed_cidrs`), всё остальное отдаёт 404.
- **Сети и rate limit.** nginx пускает к `/v1/` только адреса из `llm_allowed_cidrs` и ограничивает частоту запросов: 10 r/s, burst 20, сверх этого 429.
- **UFW и Docker.** UFW включён с политикой deny на входящие, разрешён только SSH из `common_ssh_allowed_cidrs`. Учтите важную деталь: Docker пишет свои правила iptables, и UFW **не фильтрует** опубликованные порты контейнеров. Поэтому доступ к API ограничивается в nginx, а dcgm-exporter по умолчанию слушает только `127.0.0.1`. Если нужен фильтр именно на уровне файрвола, правила надо писать в цепочку `DOCKER-USER`.

## Проверка без GPU

На машине без GPU плейбук намеренно останавливается на первой задаче роли `nvidia_driver` с сообщением `No NVIDIA GPU found`. Остальное проверяется статически:

```bash
ansible-playbook site.yml --syntax-check
```

```bash
ansible-lint --profile production
```

```bash
yamllint .
```

Шаблон `compose.yml.j2` после рендера проходит `docker compose config`. Всё перечисленное проходит без ошибок. Версии пакетов и образов сверены с репозиториями: драйвер 580-server и Fabric Manager есть для 22.04, 24.04 и 26.04, toolkit 1.20.2-1 опубликован в apt-репозитории NVIDIA. На реальном железе плейбук пока не запускался. Конфиг nginx через `nginx -t` тоже не проверялся.

## Что улучшить дальше

- **Несколько моделей и учёт клиентов.** Поставить перед vLLM шлюз вроде LiteLLM: по ключу на команду, квоты и бюджеты, роутинг по моделям и фолбэк на внешние API.
- **Большие модели.** Для моделей больше одной карты есть `llm_tensor_parallel_size`. Квантизация (AWQ, FP8) через `llm_extra_args` позволяет уместить 70B на 2×H100.
- **Кэш весов.** Веса стоит держать на отдельном NVMe-диске или во внутреннем зеркале HF/S3, чтобы новые серверы не качали их из интернета.
- **Отказоустойчивость.** Сделать несколько GPU-серверов за балансировщиком. Для вывода узла из ротации использовать `/health`, выкатывать по одному узлу (`serial: 1`).
- **Обновления драйвера.** `unattended-upgrades` может обновить userspace драйвера без перезагрузки, после чего до ребута будет `Driver/library version mismatch`. Ветку драйвера лучше обновлять только этим плейбуком, с плановой перезагрузкой.
- **Тесты роли.** Molecule-сценарий на GPU-раннере в CI.
- **Kubernetes.** Если GPU-серверов станет много, перейти на GKE/k8s с NVIDIA GPU Operator. Он сам ставит драйвер, toolkit и DCGM, а модели раскатываются через KServe или vLLM production-stack.

## Структура

```
site.yml                      # плейбук: common → nvidia_driver → nvidia_toolkit → docker → llm
group_vars/all/main.yml       # основные настройки
group_vars/all/vault.yml      # секреты (шаблон: vault.yml.example)
inventory/hosts.yml           # сервера (шаблон: hosts.yml.example)
roles/common                  # проверка ОС, базовые пакеты, UFW
roles/nvidia_driver           # драйвер R580-server, Fabric Manager (опционально), persistenced
roles/nvidia_toolkit          # NVIDIA Container Toolkit
roles/docker                  # Docker CE, daemon.json с runtime nvidia и ротацией логов
roles/llm                     # compose: vLLM + nginx + dcgm-exporter, проверка (verify.yml)
```
