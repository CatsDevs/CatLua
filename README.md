# LuaHub

Веб-редактор `.lua` файлов с синхронизацией в GitHub-репозиторий и генерацией
`loadstring`-ссылок для быстрой загрузки скриптов в Roblox Studio.

Архитектура:
- **Сайт** (`index.html`) — статика, живёт на GitHub Pages. Логин, выбор
  репозитория, список файлов, редактор Monaco, коммит через GitHub API.
- **Worker** (`worker.js`) — крошечный serverless-эндпоинт на Cloudflare
  Workers. Единственная задача — обменять OAuth `code` на `access_token`,
  потому что это единственный шаг, где нужен секретный ключ (`client_secret`),
  который нельзя хранить в браузере.

## 1. Создать GitHub OAuth App

1. GitHub → Settings → Developer settings → OAuth Apps → **New OAuth App**
2. Homepage URL: `https://<твой-юзернейм>.github.io/<репозиторий>/`
3. Authorization callback URL: тот же адрес
4. Сохрани **Client ID** и сгенерируй **Client Secret**

## 2. Задеплоить Worker (Cloudflare)

```bash
npm install -g wrangler
wrangler login

cd site
wrangler secret put GITHUB_CLIENT_ID
wrangler secret put GITHUB_CLIENT_SECRET
wrangler deploy
```

После деплоя скопируй адрес вида `https://luahub-oauth.<subdomain>.workers.dev`.

В `worker.js` поменяй `ALLOWED_ORIGIN` на реальный адрес твоего GitHub Pages
(иначе браузер заблокирует запросы по CORS).

## 3. Настроить сайт

В `index.html`, в блоке `CONFIG`, впиши:

```js
GITHUB_CLIENT_ID: "<Client ID из шага 1>",
TOKEN_EXCHANGE_URL: "https://luahub-oauth.<subdomain>.workers.dev/token",
```

> Обрати внимание: путь `/token` в Worker'е не обязателен — можно оставить
> корень воркера, тогда просто убери `/token` из `TOKEN_EXCHANGE_URL`.

## 4. Включить GitHub Pages

1. Запушь `index.html` в репозиторий (например, в ветку `main`, папку `/docs`
   или через GitHub Actions — как удобнее)
2. Repo → Settings → Pages → Source → выбери ветку/папку
3. Открой полученный `https://<юзернейм>.github.io/<репо>/`

## Как это работает

1. Пользователь жмёт «Войти через GitHub» → редирект на GitHub OAuth
2. GitHub возвращает `code` на сайт → сайт отправляет его в Worker
3. Worker меняет `code` + `client_secret` на `access_token` через GitHub API
4. Токен возвращается в браузер и хранится в `localStorage`
5. Дальше все запросы (список репо, чтение/запись файлов) идут напрямую
   из браузера в GitHub REST API с этим токеном
6. Кнопка «Loadstring» собирает ссылку вида:
   ```
   loadstring(game:HttpGet("https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<file>.lua"))()
   ```

## Ограничения текущей версии

- Токен лежит в `localStorage` — ок для личного использования, но не для
  продакшена с чужими пользователями (нет refresh-токенов, нет server-side
  сессий)
- OAuth-скоуп `repo` даёт доступ и к приватным репозиториям — если нужен
  доступ только к публичным, поменяй `scope` в `startLogin()` на `public_repo`
- Нет проверки `state` параметра при OAuth-редиректе (защита от CSRF) —
  для личного проекта не критично, но для публичного стоит добавить
