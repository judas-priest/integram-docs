# Pages

**Директория:** `pages/[db]/portal/`

Nuxt использует файловую маршрутизацию. Все страницы принимают `[db]` как динамический параметр.

| Файл | URL | Назначение | Auth |
|------|-----|-----------|------|
| `index.vue` | `/:db/portal/` | Главная страница портала | нет |
| `auth.vue` | `/:db/portal/auth` | Авторизация: ввод телефона + OTP | нет |
| `auth-link.vue` | `/:db/portal/auth-link` | Вход по персональной ссылке: обмен токена/кода (`?t=` / `?c=` от `/l/<code>`) на сессию | нет (токен в ссылке) |
| `catalog/index.vue` | `/:db/portal/catalog` | Каталог товаров/услуг | нет |
| `catalog/[slug].vue` | `/:db/portal/catalog/:slug` | Карточка товара | нет |
| `cart.vue` | `/:db/portal/cart` | Корзина | нет (guest cart) |
| `orders/index.vue` | `/:db/portal/orders` | Список заказов клиента | Portal JWT |
| `orders/[id].vue` | `/:db/portal/orders/:id` | Детали заказа | Portal JWT |
| `profile.vue` | `/:db/portal/profile` | Профиль клиента | Portal JWT |
| `documents.vue` | `/:db/portal/documents` | Документы клиента | Portal JWT |
| `metrics.vue` | `/:db/portal/metrics` | KPI / метрики клиента | Portal JWT |
| `contacts.vue` | `/:db/portal/contacts` | Контакты компании | нет |
| `gallery.vue` | `/:db/portal/gallery` | Галерея | нет |
| `wishlist.vue` | `/:db/portal/wishlist` | Избранное | нет |
| `kb/index.vue` | `/:db/portal/kb` | База знаний | нет |
| `kb/[id].vue` | `/:db/portal/kb/:id` | Статья базы знаний | нет |
| `support/index.vue` | `/:db/portal/support` | Поддержка | Portal JWT |
| `support/[id].vue` | `/:db/portal/support/:id` | Карточка обращения (тикета) | Portal JWT |
| `coming-soon.vue` | `/:db/portal/coming-soon` | Заглушка "скоро" | нет |
| `staff.vue` | `/:db/portal/staff` | Панель сотрудника: список заказов + форма создания/редактирования | `portal_staff` JWT |
| `teamchat.vue` | `/:db/portal/teamchat` | Командный чат портала | Portal JWT |
| `meta-kb/index.vue` | `/:db/portal/meta-kb` | Список топиков, просмотр дебатов | Portal JWT |
| `meta-kb/[topicId].vue` | `/:db/portal/meta-kb/:topicId` | Детали топика с дебатами | Portal JWT |
| `decisions/index.vue` | `/:db/portal/decisions` | Список решений с поиском | Portal JWT |
| `decisions/[id].vue` | `/:db/portal/decisions/:id` | Детали решения | Portal JWT |
| `chat.vue` | `/:db/portal/chat` | AI-чат портала | Portal JWT |
| `office.vue` | `/:db/portal/office` | Редирект в SPA-раздел «Подать заявку» (`#/apply`), заявка подхватывается из localStorage | нет |
| `shared/chat/[token].vue` | `/:db/portal/shared/chat/:token` | Просмотр расшаренной переписки чата по токену (minimal layout) | нет (токен в ссылке) |

## Layout

Все страницы используют layout `layouts/portal.vue`, который подключает:
- Хедер с навигацией (из конфига портала)
- ChatWidget (AI-чат)
- Футер
