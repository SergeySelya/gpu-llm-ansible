# Проверка на реальном сервере

Стенд: GCP `n1-standard-4` + Tesla T4 16 ГБ, Ubuntu 24.04.5 LTS, Secure Boot включён, vLLM v0.34.0, `Qwen/Qwen2.5-1.5B-Instruct`. Все выводы сняты со стенда как есть, заменены только имя хоста и проекта. Ключ API в выводы не попадал.

Сводка — в README, раздел «Проверено на T4».

## 1. Плейбук: полный прогон

Всё ставится с нуля одной командой, проверка в конце прошла.

```text
$ ansible-playbook site.yml
...
TASK [nvidia_driver : Show detected GPUs] **************************************
ok: [gpu-01] => 
    nvidia_driver_gpus.stdout_lines:
    - 0, Tesla T4, 580.178.04, 15360 MiB, 7.5
...
TASK [llm : Show model reply] **************************************************
ok: [gpu-01] => 
    msg: 'qwen2.5-1.5b: OK (37 tokens)'

PLAY RECAP
gpu-01           : ok=55   changed=10   unreachable=0    failed=0    skipped=6    rescued=0    ignored=0
```

## 2. Идемпотентность

Повторный прогон ничего не меняет.

```text
$ ansible-playbook site.yml   # повторно, ~1 мин
PLAY RECAP
gpu-01           : ok=53   changed=0    unreachable=0    failed=0    skipped=6    rescued=0    ignored=0
```

## 3. Драйвер и GPU

Драйвер 580 (open-модуль) работает с Secure Boot, vLLM держит модель в видеопамяти T4.

```text
$ lsb_release -ds; uname -r; mokutil --sb-state
Ubuntu 24.04.5 LTS
7.0.0-1018-gcp
SecureBoot enabled

$ nvidia-smi
Mon Nov 23 21:41:26 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.178.04             Driver Version: 580.178.04     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla T4                       Off |   00000000:00:04.0 Off |                    0 |
| N/A   52C    P0             28W /   70W |   13073MiB /  15360MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            5915      C   VLLM::EngineCore                      13070MiB |
+-----------------------------------------------------------------------------------------+

$ modinfo -F license nvidia; cat /proc/driver/nvidia/version
Dual MIT/GPL
NVRM version: NVIDIA UNIX Open Kernel Module for x86_64  580.178.04  Release Build  (dvs-builder@U22-I3-AE18-16-4)  Tue Jul  7 12:18:12 UTC 2026

$ docker run --rm --gpus all ubuntu:24.04 nvidia-smi -L
GPU 0: Tesla T4 (UUID: GPU-xxxxxxxx)
```

## 4. Контейнеры и автозапуск

Стек healthy, службы включены в автозапуск, nginx -t чистый.

```text
$ docker compose ps (llm + monitoring)
SERVICE         IMAGE                                    STATUS                    PORTS
dcgm-exporter   nvcr.io/nvidia/k8s/dcgm-exporter:4.8.4   Up 8 minutes              127.0.0.1:9400->9400/tcp
nginx           nginx:1.30-alpine                        Up 30 seconds (healthy)   80/tcp, 0.0.0.0:443->443/tcp
vllm            vllm/vllm-openai:v0.34.0                 Up 8 minutes (healthy)    
SERVICE      IMAGE                     STATUS         PORTS
grafana      grafana/grafana:13.2.6    Up 2 minutes   127.0.0.1:3000->3000/tcp
prometheus   prom/prometheus:v3.15.1   Up 2 minutes   127.0.0.1:9090->9090/tcp

$ systemctl is-enabled docker nvidia-persistenced llm-watchdog.timer
docker                 enabled active
nvidia-persistenced    static active
llm-watchdog.timer     enabled active

$ docker compose exec nginx nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## 5. API: ключ и запрос к модели

Без ключа 401, с ключом модель отвечает в формате OpenAI.

```text
$ curl -sk https://127.0.0.1/health -w '%{http_code}'
HTTP 200

$ curl -sk https://127.0.0.1/v1/models
{"error":"Unauthorized"}
HTTP 401

$ curl -sk https://127.0.0.1/v1/models -H 'Authorization: Bearer wrong'
{"error":"Unauthorized"}
HTTP 401

$ curl -sk https://127.0.0.1/v1/models -H 'Authorization: Bearer $LLM_KEY'
{
  "data": [
    {
      "id": "qwen2.5-1.5b",
      "root": "Qwen/Qwen2.5-1.5B-Instruct",
      "max_model_len": 8192
    }
  ]
}

$ curl -sk https://127.0.0.1/v1/chat/completions -H 'Authorization: Bearer $LLM_KEY' -d '{...}'
{
  "model": "qwen2.5-1.5b",
  "answer": "Ключ-значение (Key-Value) кэш - это тип кэша, который использует ключи для хранения значений. В контексте Large Language Models (LLMs), такой кэш может использоваться для быстрого доступа к уже существующим данным, предварительно обученным на больших объемах данных.\n\nВот более подробное объяснение:\n\n1. **Создание и использование:**\n   - Каждый элемент в кэше имеет уникальный ключ.\n   - Значение, соответствующее этому",
  "finish_reason": "length",
  "usage": {
    "prompt_tokens": 46,
    "total_tokens": 166,
    "completion_tokens": 120,
    "completion_tokens_details": null
  }
}
```

## 6. Запрос с клиента (ноутбук)

Клиент снаружи по HTTPS через nginx.

```text
$ curl -sk https://<сервер>/v1/chat/completions -H "Authorization: Bearer $LLM_KEY" ...   # с ноутбука, клиент 35.235.240.0/20 добавлен в llm_allowed_cidrs
{
    "id": "chatcmpl-89c507894393497e",
    "object": "chat.completion",
    "created": 1795470195,
    "model": "qwen2.5-1.5b",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "Ansible - это пакет инструментов для управления и автоматизации системами через SSH.",
                "refusal": null,
                "audio": null,
                "function_call": null,
                "reasoning": null
            },
            "logprobs": null,
            "finish_reason": "stop",
            "stop_reason": null,
            "token_ids": null,
            "routed_experts": null,
            "weight_versions": null
        }
    ],
    "service_tier": null,
    "system_fingerprint": "vllm-0.34.0-3c117dc8",
    "usage": {
        "prompt_tokens": 37,
        "total_tokens": 60,
        "completion_tokens": 23,
        "completion_tokens_details": null
    },
    "prompt_logprobs": null,
    "prompt_token_ids": null,
    "prompt_text": null,
    "kv_transfer_params": null,
    "ec_transfer_params": null,
    "metrics": null
}
```

## 7. Стриминг

Токены приходят по мере генерации (SSE).

```text
$ curl -skN ... -d '{"stream": true, ...}'
data: {"id":"chatcmpl-85fc3f37210ac2a5","object":"chat.completion.chunk","created":1795470090,"model":"qwen2.5-1.5b","choices":[{"index":0,"delta":{"role":"assistant","content":""},"logprobs":null,"finish_reason":null}],"prompt_token_ids":null,"prompt_text":null}

data: {"id":"chatcmpl-85fc3f37210ac2a5","object":"chat.completion.chunk","created":1795470090,"model":"qwen2.5-1.5b","choices":[{"index":0,"delta":{"content":"Т"},"logprobs":null,"finish_reason":null,"token_ids":null}],"kv_transfer_params":null}

data: {"id":"chatcmpl-85fc3f37210ac2a5","object":"chat.completion.chunk","created":1795470090,"model":"qwen2.5-1.5b","choices":[{"index":0,"delta":{"content":"ри"},"logprobs":null,"finish_reason":null,"token_ids":null}],"kv_transfer_params":null}
```

## 8. Защита: allowlist, rate limit, закрытые порты

Чужая сеть получает 403, лишние запросы 429, vLLM наружу не торчит.

```text
$ curl с адреса вне llm_allowed_cidrs (35.235.240.x):
/v1/models -> HTTP 403
/metrics -> HTTP 403
nginx: access forbidden by rule, client: 35.235.240.114

$ for i in 1..25: curl /v1/models
     22 200
      3 429

$ curl / и /docs -> 404; ss -ltn
/ -> HTTP 404
/docs -> HTTP 404
слушающие порты:
0.0.0.0:22
0.0.0.0:443
127.0.0.1:3000
127.0.0.1:9090
127.0.0.1:9400
[::]:22
```

## 9. Мониторинг

Метрики GPU собираются, Prometheus видит все цели, алерты загружены.

```text
$ curl -s 127.0.0.1:9400/metrics | grep -E 'GPU_UTIL|FB_USED|GPU_TEMP|POWER_USAGE'
DCGM_FI_DEV_FB_USED{gpu="0",...} 13072
DCGM_FI_DEV_GPU_TEMP{gpu="0",...} 52
DCGM_FI_DEV_GPU_UTIL{gpu="0",...} 0
DCGM_FI_DEV_POWER_USAGE{gpu="0",...} 28.624

$ curl 127.0.0.1:9090/api/v1/targets
dcgm	up	http://dcgm-exporter:9400/metrics
prometheus	up	http://localhost:9090/metrics
vllm	up	http://vllm:8000/metrics

$ curl 127.0.0.1:9090/api/v1/rules
LLMApiDown	ok	inactive
GPUExporterDown	ok	inactive
GPUXidErrors	ok	inactive
GPUOverheat	ok	inactive
LLMQueueGrowing	ok	inactive
LLMKVCacheFull	ok	inactive
```

## 10. Перезагрузка сервера

После ребута всё поднимается само, без Ansible.

```text
$ sudo reboot; ...; docker inspect (время старта контейнеров)
boot: 2026-11-23 21:51:48
/llm-nginx-1 started=2026-11-23T21:52:12.988686211Z health=healthy restarts=0
/monitoring-grafana-1 started=2026-11-23T21:52:13.002496843Z health=- restarts=0
/monitoring-prometheus-1 started=2026-11-23T21:52:13.004720615Z health=- restarts=0
/llm-vllm-1 started=2026-11-23T21:52:13.005863743Z health=healthy restarts=0
/llm-dcgm-exporter-1 started=2026-11-23T21:52:12.987749685Z health=- restarts=0
INFO:     Application startup complete.
NEXT                           LEFT LAST PASSED UNIT               ACTIVATES
Mon 2026-11-23 21:56:48 UTC 2min 1s -         - llm-watchdog.timer llm-watchdog.service

$ ansible-playbook site.yml --tags verify
TASK [llm : Show model reply] **************************************************
ok: [gpu-01] => 
    msg: 'qwen2.5-1.5b: OK (37 tokens)'
PLAY RECAP
gpu-01           : ok=13   changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## 11. Watchdog

Зависший vLLM (процесс жив, но не отвечает) перезапускается автоматически.

```text
$ kill -STOP <pid vLLM API>; следим за health
21:55:06 SIGSTOP vllm api pid 1807
21:55:06 healthy restarts=0 started=2026-11-23T21:52:13.005863743Z
21:56:02 unhealthy restarts=0 started=2026-11-23T21:52:13.005863743Z
21:57:17 starting restarts=0 started=2026-11-23T21:57:14.942543383Z
21:58:13 healthy restarts=0 started=2026-11-23T21:57:14.942543383Z
--- watchdog journal
-- Boot 2822418dd27546469394a78d920e69a0 --
2026-11-23T21:57:04+00:00 gpu-01 llm-watchdog[4509]: restarting unhealthy container /llm-vllm-1 (project llm)

health after recovery: HTTP 200
```

## 12. Бенчмарк на Tesla T4

vllm bench serve: 300 запросов, вход 512 / выход 128 токенов, 16 параллельно.

```text
============ Serving Benchmark Result ============
Successful requests:                     300       
Failed requests:                         0         
Maximum request concurrency:             16        
Benchmark duration (s):                  90.86     
Total input tokens:                      162299    
Total generated tokens:                  38400     
Request throughput (req/s):              3.30      
Output token throughput (tok/s):         422.62    
Peak output token throughput (tok/s):    704.00    
Peak concurrent requests:                32.00     
Total token throughput (tok/s):          2208.85   
---------------Time to First Token----------------
Mean TTFT (ms):                          872.42    
Median TTFT (ms):                        1017.01   
P50 TTFT (ms):                           1017.01   
P95 TTFT (ms):                           1231.53   
P99 TTFT (ms):                           1915.77   
-----Time per Output Token (excl. 1st token)------
Mean TPOT (ms):                          30.90     
Median TPOT (ms):                        30.20     
P50 TPOT (ms):                           30.20     
P95 TPOT (ms):                           35.77     
P99 TPOT (ms):                           36.96     
----------------End-to-end Latency----------------
Mean E2EL (ms):                          4797.16   
Median E2EL (ms):                        4807.10   
P50 E2EL (ms):                           4807.10   
P95 E2EL (ms):                           4906.54   
P99 E2EL (ms):                           5293.70   
==================================================
```
