# TASKFLOW — Vercel + Cloudflare D1

Фронтенд React/Vite на Vercel; `/api/proxy` — Vercel Function, которая пересылает запросы в Cloudflare Worker; Worker хранит общие проекты, задачи и комментарии в Cloudflare D1. Данные обновляются у всех пользователей каждые 3 секунды (polling, **не WebSocket**).

## Настройка (один раз)

Требуются аккаунты Cloudflare и Vercel, Node.js 20+ и терминал. **Просто загрузить ZIP на Vercel без настройки D1 нельзя.**

1. В каталоге `cloudflare` выполните `npx wrangler login`, затем `npx wrangler d1 create taskflow-db`.
2. Вставьте полученный `database_id` вместо `REPLACE_WITH_D1_DATABASE_ID` в `cloudflare/wrangler.toml`.
3. Выполните `npx wrangler d1 execute taskflow-db --remote --file=./schema.sql` (из каталога `cloudflare`).
4. Придумайте длинный случайный секрет, например `openssl rand -hex 32`. Установите его для Worker командой `npx wrangler secret put API_SECRET` (введите секрет в приглашение).
5. Из каталога `cloudflare` выполните `npx wrangler deploy`. Скопируйте URL вида `https://taskflow-api.<subdomain>.workers.dev`.
6. Загрузите **корень проекта** в GitHub и импортируйте репозиторий в Vercel. Framework: Vite, Build: `npm run build`, Output: `dist`.
7. В Vercel → Project → Settings → Environment Variables добавьте **серверные** переменные `CLOUDFLARE_WORKER_URL` (адрес из шага 5) и `TASKFLOW_API_SECRET` (точно тот же секрет из шага 4). Затем выполните Redeploy.

Для локального запуска Vercel Functions используйте `npx vercel dev` после настройки переменных. `npm run dev` запускает только Vite и не включает `/api/proxy`.

## Важно о доступе

Как и в исходном проекте с общими Firestore-правилами, здесь **нет регистрации, ролей и приватных проектов**. Любой, кто откроет публичный сайт, увидит и сможет изменить все проекты. Секрет защищает Worker от прямых запросов, но **не ограничивает посетителей Vercel-сайта**. Не храните конфиденциальные данные до добавления аутентификации и правил доступа.

В отличие от исходного Firebase-проекта, это синхронизация опросом каждые 3 секунды, а не мгновенная подписка. Вся работа с D1 происходит в Cloudflare Worker, не в браузере и не напрямую из Vercel.

## Новая тема и цвета карточек

Интерфейс выполнен в чёрно-бело-оранжевой гамме с умеренным оранжевым свечением. Откройте задачу или создайте новую → **Цвет задачи** → выберите цвет → **Сохранить задачу**. Цвет хранится в D1 и виден всем участникам после синхронизации.

### Если вы уже развернули старую версию базы

Не выполняйте удаление базы! Добавьте столбец командой из папки `cloudflare`:

```bash
npx wrangler d1 execute taskflow-db --remote --command="ALTER TABLE tasks ADD COLUMN color TEXT NOT NULL DEFAULT 'orange';"
```

Для **новой** базы достаточно `schema.sql`, как в инструкции выше. Не запускайте ALTER TABLE второй раз: столбец уже будет существовать.

### Подробно: куда загружать

1. Распакуйте ZIP. Установите Node.js 20+ и Git. Откройте терминал в папке `taskflow/cloudflare`.
2. Создайте бесплатный аккаунт на https://dash.cloudflare.com/ и войдите: `npx wrangler login`.
3. Создайте D1: `npx wrangler d1 create taskflow-db`. Скопируйте `database_id` из ответа в `cloudflare/wrangler.toml` вместо `REPLACE_WITH_D1_DATABASE_ID`.
4. Выполните `npx wrangler d1 execute taskflow-db --remote --file=./schema.sql`.
5. Создайте секрет: `openssl rand -hex 32` (либо используйте генератор паролей и длинную случайную строку). Сохраните значение в менеджере паролей.
6. Выполните `npx wrangler secret put API_SECRET`, вставьте секрет и нажмите Enter. Затем `npx wrangler deploy`. Сохраните адрес Worker `https://...workers.dev`.
7. Создайте репозиторий на https://github.com/new и загрузите **содержимое папки `taskflow`** (так, чтобы `package.json` лежал в корне репозитория). Не загружайте секреты или `.env` в GitHub.
8. Откройте https://vercel.com/new, войдите через GitHub, выберите репозиторий и нажмите Import. Framework Preset — **Vite**, Root Directory — **./**, Build Command — **npm run build**, Output Directory — **dist**.
9. В настройках проекта Vercel → Settings → Environment Variables создайте `CLOUDFLARE_WORKER_URL` со значением URL Worker (без завершающего `/`) и `TASKFLOW_API_SECRET` со значением секрета из шага 5. **Не** добавляйте префикс `VITE_` к этим переменным.
10. Перейдите Deployments → Redeploy (сборка должна пройти после сохранения переменных). Откройте ссылку `*.vercel.app`, создайте проект, добавьте задачу и комментарий. Проверьте обновление во второй вкладке браузера.

**Важно:** ссылка-приглашение не является механизмом авторизации. На этой версии любой посетитель публичного сайта может просматривать и редактировать все проекты. Для приватной работы необходимо добавить учётные записи и проверку прав на сервере.
