# Контракты VolleyBOT

- [OpenAPI 3.1](../api/openapi/volleybot.v1.yaml) описывает внешний REST API `users` и `games`.
- [users.proto](../api/proto/volleybot/users/v1/users.proto), [games.proto](../api/proto/volleybot/games/v1/games.proto) и [common.proto](../api/proto/volleybot/common/v1/common.proto) описывают внутренние вызовы gRPC.
- [Каталог событий Kafka](events.md) описывает публикацию изменений для межсервисной обработки.

Контракты следуют [дорожной карте](../README.md#этапы), [правилам клуба](domain.md) и [архитектуре](architecture.md). Поле `x-stage` в OpenAPI указывает этап появления операции. К вехе v0.1.0 игрок может через REST посмотреть игры, записаться, отменить свою запись и посмотреть свои записи. С этапа 16 бот вызывает те же сценарии в `games` по gRPC.

## REST и gRPC

REST доступен через Nginx по одному адресу с префиксом `/api/v1`. Маршруты `/auth`, `/me` и `/users` принадлежат `users`. Маршруты `/venues`, `/halls`, `/game-formats`, `/games` и `/series` принадлежат `games`. Маршрутизация учитывает границы сегментов пути. Идентификаторы непрозрачны. Метки времени передаются в RFC 3339 и возвращаются в UTC. Местные даты серии интерпретируются в часовом поясе серии. Стоимость игры передаётся в копейках.

REST передаёт access JWT в заголовке `Authorization: Bearer <token>`. `x-access` описывает минимальную роль. Авторизация конкретной операции учитывает актуальный бан, роль вызывающего и владельца ресурса. Чтение своих данных и сведений о бане доступно забаненному пользователю. Изменение данных и запись на игру требуют отсутствия активного бана. Публичные ответы об играх содержат только числа участников. Список с именами возвращается владельцу-модератору или администратору. Игрок видит свой статус записи и свою команду при её наличии.

gRPC доступен по доверенному внутреннему каналу. Сервисы проверяют вызывающий сервис и разрешённые для него операции. Для действий бота от имени игрока `telegram` получает `user_id` из привязки по `chat_id`, полученному от Telegram, и передаёт его в `games`. `games` проверяет актуальные роль и бан в `users`. Методы чтения привязок, профилей и настроек возвращают только поля, нужные для сценария. Вызывающий сервис задаёт дедлайн. Повтор записи возвращает существующую активную запись. Отмена адресуется по `registration_id`, чтобы запоздалый повтор отмены не затронул новую запись игрока на ту же игру.

## Ошибки

REST возвращает JSON с обязательными `code` и `message`. Клиенты ветвятся по `code`. `message` предназначен для человека. gRPC возвращает канонический статус и содержит `volleybot.common.v1.ErrorDetail` в `google.rpc.Status.details` для прикладных ошибок.

| HTTP | gRPC | Возможные коды VolleyBOT |
| --- | --- | --- |
| 400 | `INVALID_ARGUMENT` | `invalid_argument`, `invalid_cursor` |
| 401 | `UNAUTHENTICATED` | `unauthenticated`, `invalid_credentials`, `invalid_refresh_token` |
| 403 | `PERMISSION_DENIED` | `permission_denied`, `user_banned` |
| 404 | `NOT_FOUND` | `not_found` |
| 409 | `ALREADY_EXISTS` | `email_in_use`, `hall_name_in_use`, `telegram_already_linked` |
| 409 | `FAILED_PRECONDITION` | `hall_time_conflict`, `registration_not_open`, `game_started`, `game_cancelled`, `invalid_state`, `invalid_link_token` |
| 429 | `RESOURCE_EXHAUSTED` | `rate_limited` |
| 500 | `INTERNAL` | `internal` |
| 503 | `UNAVAILABLE` | `unavailable` |
| 504 | `DEADLINE_EXCEEDED` | `deadline_exceeded` |

У `invalid_argument` могут быть ошибки полей в `violations`. У `hall_time_conflict` каждый элемент `conflicts` описывает конфликтующую игру и зал. Изменение серии возвращает список конфликтов. У `registration_not_open` передаётся `registration_opens_at`. Забаненному игроку `user_banned` возвращает его причину и срок в `ban`. Для бессрочного бана REST возвращает `ends_at: null`, protobuf оставляет поле пустым.

```json
{
  "code": "registration_not_open",
  "message": "Запись ещё не открыта",
  "registration_opens_at": "2026-12-01T16:00:00Z"
}
```

## Пагинация

Клиент запрашивает `limit` от 1 до 100, по умолчанию 20. Ответ содержит `items` и, если есть следующая страница, непрозрачный `next_cursor`. Клиент передаёт его как `cursor`, сохраняя фильтры и область доступа. Курсор другого списка, фильтра или пользователя отклоняется с `invalid_cursor`. Для списка предстоящих игр нижняя граница времени начала фиксируется в курсоре по первому запросу.

Игры отсортированы по возрастанию `(starts_at, game_id)`, свои записи — по убыванию `(game.starts_at, registration_id)`, состав игры — по возрастанию `seq`, серии, площадки и залы — по возрастанию идентификатора. Кандидаты на напоминание упорядочены по `(game.starts_at, game_id, registration_id)`. Сортировка стабильна для неизменных данных. Между страницами видны текущие данные.
