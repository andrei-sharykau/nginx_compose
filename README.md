# nginx + certbot (Docker Compose)

Playbook выкладывает `docker-compose.yml` (nginx + certbot), дефолтный `nginx.conf` с хоста и для каждого домена проходит цепочку: HTTP (ACME) → выпуск LE-сертификата → HTTPS + redirect + robots (+ optional basic auth).

## Требования

- Docker и Compose plugin (`docker compose`) на целевом хосте
- Порты 80/443 свободны и доступны из интернета (ACME HTTP-01)
- DNS доменов указывает на хост

## Запуск

```bash
ansible-playbook nginx_compose/main.yml -l <host> -e @nginx_compose_vars.yml
```

## Пример `nginx_compose_vars.yml`

```yaml
nginx_compose_dir: /opt/nginx-compose
nginx_compose_email: admin@example.com
nginx_compose_staging: false   # true — Let's Encrypt staging

nginx_compose_domains:
  - name: example.com
    aliases:
      - www.example.com
    basic_auth:
      enabled: true
      realm: Restricted
      users:
        - name: user1
          password: "change-me"
    robots: |
      User-agent: *
      Disallow: /
    locations: |
      location / {
          proxy_pass http://172.17.0.1:8080;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }

  - name: app.example.com
    robots: |
      User-agent: *
      Allow: /
    locations: |
      location / {
          proxy_pass http://172.17.0.1:9000;
          proxy_set_header Host $host;
      }
```

- `robots` — сырой текст `robots.txt` (если не задан — `Disallow: /`)
- `locations` — сырой nginx-блок; попадает **только в HTTPS** `server` после выпуска сертификата
- `basic_auth` — опционально; включается **только на `:443` после TLS**. На HTTP/ACME-стадии auth нет. `/robots.txt` без auth

## Алгоритм per-domain

1. HTTP vhost: ACME + stub `404` (без auth, без `locations`)
2. `nginx -t` + reload
3. `certbot certonly` (webroot)
4. HTTPS vhost: redirect 80→443, TLS, `locations`, при необходимости basic auth
5. `nginx -t` + reload

## ФС на хосте

```
/opt/nginx-compose/
├── docker-compose.yml
├── conf/nginx.conf          # создаётся один раз, дальше не перезаписывается
├── conf/conf.d/<domain>.conf
├── html/<domain>/robots.txt
├── auth/<domain>.htpasswd
├── certbot/conf/            # /etc/letsencrypt
├── certbot/www/             # ACME webroot
└── logs/
```
