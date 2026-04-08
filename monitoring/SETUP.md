# Zabbix Monitoring Setup for Coolify

## 1. Import custom template

**Configuration → Templates → Import**
→ выбери файл `zabbix-template-coolify-api.yaml`

---

## 2. Создай хост для Coolify API

**Configuration → Hosts → Create host**

| Поле | Значение |
|---|---|
| Host name | `Coolify` |
| Interfaces | нет (HTTP monitoring, агент не нужен) |
| Templates | `Coolify by HTTP` |

После создания хоста задай макросы (**Macros** tab):

| Макрос | Значение |
|---|---|
| `{$COOLIFY_URL}` | `https://coolify.твой-домен.com` |
| `{$COOLIFY_API_TOKEN}` | токен из Coolify UI → Profile → API Tokens |

---

## 3. Создай хост для сервера (Agent 2)

**Configuration → Hosts → Create host**

| Поле | Значение |
|---|---|
| Host name | `Coolify Server` |
| Interfaces → Agent | IP сервера, порт `10050` |
| Templates | см. ниже |

### Шаблоны для подключения (встроенные, не нужно импортировать):

| Шаблон | Что мониторит |
|---|---|
| `Linux by Zabbix agent 2` | CPU, RAM, диск, сеть, процессы |
| `Docker by Zabbix agent 2` | все контейнеры: статус, CPU, RAM, рестарты |

> **Docker template требует:** на хосте добавить в `/etc/zabbix/zabbix_agent2.conf`:
> ```
> Plugins.Docker.Endpoint=unix:///var/run/docker.sock
> ```
> И добавить пользователя zabbix в группу docker:
> ```bash
> sudo usermod -aG docker zabbix
> sudo systemctl restart zabbix-agent2
> ```

---

## 4. Проверка

- **Monitoring → Latest data** → фильтр по хосту `Coolify` — должны появиться данные через ~2 минуты
- **Monitoring → Problems** — триггеры сработают если API недоступен или есть упавшие приложения
- **Monitoring → Dashboards** → `Coolify Overview` — автоматически создаётся при импорте шаблона
