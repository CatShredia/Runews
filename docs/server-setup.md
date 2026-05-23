# Настройка сервера на хостинге Timeweb для публичного сайта (Runews)

Инструкция ориентирована на **VPS/VDS** или **облачный сервер Timeweb** с **Linux**, куда вы ставите **Docker** и сами настраиваете **Nginx/Certbot** (или аналог) для HTTPS. Для проекта **Runews** ниже учтены `docker-compose.yml` и корневой `README.md`.

> **Важно:** обычный **виртуальный хостинг** Timeweb (общий PHP-хостинг без полного root и без вашего Docker) **не подходит** под текущий Docker-стек Runews. Нужен именно **VPS/VDS** или сервер в **Timeweb Cloud**.

---

## 1. Что заказать в Timeweb

| Вариант | Где | Подходит для Runews |
|--------|-----|---------------------|
| **VPS/VDS** (классический продукт) | Личный кабинет на [timeweb.com](https://timeweb.com) → раздел VPS/VDS | Да: полный SSH, своя ОС, ставите Docker. |
| **Облачные серверы** | [Timeweb Cloud](https://timeweb.cloud) | Да: то же самое по сути; удобны сетевой **Firewall** в панели и масштабирование. |

При заказе выберите **Ubuntu 22.04/24.04 LTS** (или другую поддерживаемую вами ОС — дальнейшие команды ниже для Debian/Ubuntu).

**Ресурсы на старт:** для Docker + PostgreSQL + API + веб-контейнера обычно разумный минимум — **1–2 vCPU, 2–4 GB RAM**, диск **от 20 GB** с запасом под БД и загрузки (`api_uploads`).

После создания сервера в панели найдите **публичный IPv4** (и при наличии **IPv6**) сервера — они понадобятся для DNS.

---

## 2. Доступ по SSH (первый вход)

1. В панели Timeweb откройте карточку сервера: там указаны **IP**, логин (часто `root`), **пароль** или подсказка, как задать/сбросить пароль.
2. Подключение с вашего ПК:

```bash
ssh root@IP_ВАШЕГО_СЕРВЕРА
```

3. Рекомендуется сразу добавить **SSH-ключ** в `~/.ssh/authorized_keys` и по возможности отключить вход по паролю для root (после проверки ключа) в `/etc/ssh/sshd_config`, затем `systemctl restart ssh`.

Справка Timeweb Cloud по SSH: [Как пользоваться SSH](https://timeweb.cloud/tutorials/linux/kak-polzovatsya-ssh).

---

## 3. Обновление системы и пользователь с sudo

```bash
apt update && apt upgrade -y
```

Создайте отдельного пользователя с `sudo` и дальше по возможности работайте под ним, а не постоянно под root.

---

## 4. Сетевой доступ: файрвол Timeweb Cloud и файрвол в ОС

### Timeweb Cloud (панель)

Если сервер в **Timeweb Cloud**, проверьте раздел **«Сети» → «Firewall»** (или привязку группы правил к серверу). Если включён **режим whitelist** (разрешающий список), **явно разрешите** входящий трафик как минимум на порты:

- **22** — SSH  
- **80** — HTTP (нужен для редиректа и выпуска Let’s Encrypt по HTTP-01)  
- **443** — HTTPS  

Иначе трафик может блокироваться **до** вашей виртуальной машины, и не поможет даже `ufw` внутри ОС.

Документация: [Управление файрволом (Timeweb Cloud)](https://timeweb.cloud/docs/firewall/upravlenie-fajrvolom).

### Классический VPS Timeweb и `ufw` в Ubuntu

На самой машине удобно включить **ufw**:

```bash
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
ufw status
```

При необходимости ограничьте SSH по IP: `ufw allow from ВАШ_IP to any port 22`.

Порт **55432** с PostgreSQL из `docker-compose.yml` не должен быть доступен из интернета — не открывайте его в панели и в `ufw` для `0.0.0.0/0`.

---

## 5. Домен и DNS в Timeweb

### Домен зарегистрирован или обслуживается в Timeweb

В панели: раздел **«Домены»** → нужный домен → **DNS / ресурсные записи**. Создайте:

- **A** для `@` (или для нужного поддомена) → **IPv4** вашего VPS/облака.  
- При использовании IPv6 — **AAAA** на IPv6 сервера.  
- Для `www` часто делают **A**/`AAAA` на тот же IP или **CNAME** на основное имя — по вашей схеме.

Справка: [Настройка DNS-записей (справочный центр Timeweb)](https://timeweb.com/ru/docs/domeny/resursnye-zapisi-domena-dns-zapisi/nastrojka-dns-zapisej).

### Домен у другого регистратора

Либо укажите у регистратора **NS-серверы Timeweb** и зону ведите в панели Timeweb, либо оставьте DNS у регистратора и создайте там записи **A**/**AAAA** на IP сервера Timeweb.

Общая статья: [Как привязать домен к VPS/VDS](https://timeweb.com/ru/community/articles/kak-prikrepit-domen-k-vds).

Проверка с вашего ПК:

```bash
dig +short your-domain.com A
```

---

## 6. HTTPS (TLS)

Публичный сайт лучше сразу отдавать по **HTTPS**:

1. **Let’s Encrypt + Certbot** на сервере + **Nginx** (или **Caddy**) как reverse proxy: TLS на 443, прокси на `http://127.0.0.1:80`, куда слушает контейнер **web** из compose.  
2. Либо **Caddy** с автоматическими сертификатами.

Условия: домен уже **резолвится** на IP сервера, порт **80** доступен с интернета для проверки Let’s Encrypt.

---

## 7. Запуск Runews через Docker на сервере Timeweb

> **Два сервиса на одном VPS** (catshredia.ru + runews.catshredia.ru): полная инструкция — [catshredias-blog/docs/deploy-vps.md](../../catshredias-blog/docs/deploy-vps.md).

1. Установите **Docker Engine** и плагин **Compose** по официальной инструкции Docker для вашей версии Ubuntu.  
2. Склонируйте репозиторий на сервер или загрузите файлы проекта.  
3. В корне создайте `.env` (переменные как в корневом [README.md](../README.md)). Для продакшена:

   - **`PUBLIC_URL`** = `https://runews.catshredia.ru` (или ваш поддомен), **без** завершающего `/`.  
   - Надёжные **`POSTGRES_PASSWORD`** и **`JWT_KEY`**.  
   - Реальный **SMTP** для почты (Mailpit в compose — для разработки).

4. Запуск:

```bash
docker compose up -d --build
```

Контейнер **web** слушает **`127.0.0.1:8080`** на хосте. Хостовый **Nginx** на `:443` проксирует `runews.catshredia.ru` → `http://127.0.0.1:8080`. В `.env` **`PUBLIC_URL`** должен совпадать с HTTPS-URL в браузере.

**Рекомендации:**

- Закройте доступ к порту PostgreSQL **55432** с интернета (и в панели Timeweb Cloud Firewall, и в `ufw`).  
- Настройте **бэкапы** томов `postgres_data` и `api_uploads`.  
- Логи: `docker compose logs -f`.

---

## 8. Обновление приложения

```bash
cd /path/to/Coursework
git pull
docker compose up -d --build
```

Миграции БД — по процедуре из [README_for_API.md](../CatshrediasNewsAPI/README_for_API.md), если проект её описывает.

---

## 9. Контрольный чеклист (Timeweb + Runews)

- [ ] Заказан **VPS/VDS** или **Timeweb Cloud**, не shared-хостинг без Docker.  
- [ ] В панели **Timeweb Cloud Firewall** (если используется) открыты **22, 80, 443** для входящих.  
- [ ] В ОС настроен **ufw** (или эквивалент), лишние порты закрыты.  
- [ ] В DNS (панель Timeweb или регистратор) записи **A**/**AAAA** указывают на сервер.  
- [ ] Настроен **HTTPS**, с **80** редирект на **443**.  
- [ ] **`PUBLIC_URL`** в `.env` совпадает с реальным URL сайта.  
- [ ] Секреты только на сервере; `.env` не в git, права `chmod 600`.  
- [ ] Бэкапы БД и загрузок.

---

## Ссылки

**Timeweb**

- [Справочный центр Timeweb (домены, DNS, хостинг)](https://timeweb.com/ru/docs/)  
- [Timeweb Cloud — документация (файрвол, сети)](https://timeweb.cloud/docs/)  
- [Настройка DNS-записей](https://timeweb.com/ru/docs/domeny/resursnye-zapisi-domena-dns-zapisi/nastrojka-dns-zapisej)  
- [Привязка домена к VPS/VDS](https://timeweb.com/ru/community/articles/kak-prikrepit-domen-k-vds)  
- [Управление файрволом (Timeweb Cloud)](https://timeweb.cloud/docs/firewall/upravlenie-fajrvolom)  
- [Как пользоваться SSH (Timeweb Cloud)](https://timeweb.cloud/tutorials/linux/kak-polzovatsya-ssh)

**Репозиторий Runews**

- [README.md](../README.md) — переменные окружения и общий запуск  
- [README_for_API.md](../CatshrediasNewsAPI/README_for_API.md)  
- [README_for_Blazor.md](../CatshrediasNews.Client/README_for_Blazor.md)

Команды установки Docker, Certbot и примеры конфигов Nginx/Caddy меняются со временем — сверяйтесь с актуальной документацией инструментов и версией ОС на сервере.
