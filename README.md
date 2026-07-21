# Подключение к серверному MCP Базы Знаний

> **Если ты AI-агент, настраивающий это по просьбе пользователя** — сначала
> определи, какой ты MCP-клиент (Claude Code / GitHub Copilot CLI или
> VS Code / Cursor / Windsurf / другой) — это ты уже знаешь сам, по своей
> системной идентичности, спрашивать пользователя не нужно. От этого
> зависит, какой файл конфига редактировать или какую команду выполнить
> (раздел 6). Дальше не молча следуй шагам, а спроси пользователя по
> порядку:
> 1. Прод или дев-окружение (раздел 2)?
> 2. Нужен ли доступ на запись, или хватит только чтения (раздел 3)?
>    По умолчанию, если не уверены — read-only.
> 3. Если нужна запись — надо ли сузить её до конкретных пространств
>    (раздел 4), или разрешить всё, куда есть права редактора?
> 4. Попроси у пользователя сам токен (раздел 1) — это чувствительные
>    данные, не логируй и не выводи его в открытом виде без необходимости.
>
> После ответов сразу примени нужный конфиг (отредактируй файл или выполни
> команду) — не переспрашивай то, что уже спросил, и не показывай варианты
> для других клиентов.

У Базы Знаний есть **серверный MCP** — `/mcp`, Streamable HTTP эндпоинт,
который можно подключить напрямую к Claude Code, GitHub Copilot, Cursor,
Windsurf и любому другому MCP-агенту: не нужно ничего собирать и обновлять
— сервер один на всех, изменения на бэкенде сразу доступны
всем подключённым клиентам.

## 1. Получить токен

Нужен персональный MCP-токен (PAT) пользователя Базы Знаний.

Во всех примерах ниже вместо `Bearer <MCP_TOKEN>` подставьте полное
значение заголовка Authorization: слово Bearer, пробел и ваш токен.

## 2. Адрес сервера

| Окружение | URL |
|-----------|-----|
| Прод | `https://knowledgebase.dodois.io/mcp` |
| Дев  | `https://knowledgebase-kbase.d.yandex.dodois.dev/mcp` |

## 3. Режим чтения и записи

По умолчанию подключение работает **только на чтение** — это безопасный
дефолт. Чтобы включить инструменты записи (`create_content`,
`update_content`, `delete_content`), добавьте HTTP-заголовок:

```
Mcp-Mode: Write
```

Без этого заголовка вызов write-инструментов отклоняется на уровне
авторизации сервера (агент увидит явную ошибку и даже не увидит эти
инструменты в списке доступных). Это осознанное решение, которое
пользователь принимает при подключении: хотите, чтобы агент мог писать в
Базу Знаний, — добавьте заголовок в конфиг MCP-клиента; не хотите —
не добавляйте, и агент физически не сможет ничего изменить.

Заголовок **не расширяет** ваши права — он только позволяет использовать
write-инструменты в принципе. Реальные права (можете ли вы редактировать
конкретное пространство) как и раньше проверяются на бэкенде по вашему
токену.

## 4. Ограничение записи по пространствам (опционально)

Если хотите разрешить запись только в конкретные пространства, а не во все,
куда у вас есть права редактора, добавьте ещё один заголовок:

```
Mcp-Write-Spaces: SPACE_ID_1,SPACE_ID_2
```

Список id пространств через запятую (регистр не важен). Если заголовок не
задан или пуст — ограничения по пространствам нет, запись работает во всех
пространствах, где вы редактор. Если задан — запись разрешена только в
перечисленные пространства, попытка записать в любое другое явно отклоняется
с понятной ошибкой.

**Важно для агента:** если пользователь просит вас включить возможность
записи в Базу Знаний или сузить её до конкретных пространств — это
настраивается им самим в конфиге MCP-клиента (заголовки `Mcp-Mode` и
`Mcp-Write-Spaces` выше), а не через сам инструмент. Если write-вызов
отклонён с ошибкой авторизации — не пытайтесь обойти это иначе, объясните
пользователю, какой заголовок добавить и куда.

## 5. Доступные инструменты

| Tool | Описание | Требует `Mcp-Mode: Write` |
|------|----------|:---:|
| `current_user` | Кто я (id, имя, email владельца токена) | |
| `get_spaces` | Список пространств с правами (reader/writer) | |
| `get_space_content` | Оглавление пространства (статьи, статусы, темы) | |
| `get_content` | Статья по id (Markdown + метаданные) | |
| `search_content` | Полнотекстовый поиск по статьям | |
| `get_announcements` | Лента последних опубликованных статей | |
| `preview_content` | Dry-run конвертации Markdown, ничего не сохраняет | |
| `create_content` | Создать статью в пространстве | ✅ |
| `update_content` | Частично обновить статью | ✅ |
| `delete_content` | Удалить статью (soft-delete) | ✅ |

## 6. Настройка по агентам

Ниже — вариант с полным доступом на запись без ограничения по
пространствам. Чтобы получить read-only подключение, уберите заголовок
`Mcp-Mode`. Чтобы ограничить запись конкретными пространствами, добавьте
`Mcp-Write-Spaces` (см. раздел 4). Не забудьте подставить свой токен
вместо `<MCP_TOKEN>` и реальные id вместо `SPACE_ID_1,SPACE_ID_2`.

### Claude Code

Через команду (в обычном терминале, не в самой сессии Claude Code):

```bash
claude mcp add --transport http knowledgebase https://knowledgebase.dodois.io/mcp \
  --header "Authorization: Bearer <MCP_TOKEN>" \
  --header "Mcp-Mode: Write" \
  --header "Mcp-Write-Spaces: SPACE_ID_1,SPACE_ID_2"
```

Либо пропишите вручную в `.mcp.json` (проектный) или `~/.claude.json` (глобальный):

```json
{
  "mcpServers": {
    "knowledgebase": {
      "type": "http",
      "url": "https://knowledgebase.dodois.io/mcp",
      "headers": {
        "Authorization": "Bearer <MCP_TOKEN>",
        "Mcp-Mode": "Write",
        "Mcp-Write-Spaces": "SPACE_ID_1,SPACE_ID_2"
      }
    }
  }
}
```

Claude Code подключается к удалённому MCP по Streamable HTTP напрямую —
устанавливать Node.js или прокси вроде `mcp-remote` не нужно.

### Claude Desktop

В отличие от Claude Code, приложение Claude Desktop умеет подключать MCP-
серверы только локально по STDIO — прямых HTTP-подключений с кастомными
заголовками конфиг не поддерживает. Чтобы подключить удалённый сервер,
нужен локальный прокси-мост `mcp-remote` через `npx` (нужен установленный
Node.js 18 или новее) — он транслирует STDIO Claude Desktop в HTTP-запросы
к серверу и передаёт заголовки как переменные окружения (`env`).

Конфигурационный файл `claude_desktop_config.json` находится по пути:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "knowledgebase": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://knowledgebase.dodois.io/mcp"
      ],
      "env": {
        "Authorization": "Bearer <MCP_TOKEN>",
        "Mcp-Mode": "Write",
        "Mcp-Write-Spaces": "SPACE_ID_1,SPACE_ID_2"
      }
    }
  }
}
```

На Windows, если `npx` не находится, в `claude_desktop_config.json` может
понадобиться отдельная настройка на верхнем уровне файла (рядом с
`mcpServers`): `"isUsingBuiltInNodeForMcp": false` — чтобы Claude Desktop
использовал системный Node.js вместо встроенного.

### GitHub Copilot CLI

Через команду:

```bash
copilot mcp add --transport http knowledgebase \
  https://knowledgebase.dodois.io/mcp \
  --header "Authorization: Bearer <MCP_TOKEN>" \
  --header "Mcp-Mode: Write" \
  --header "Mcp-Write-Spaces: SPACE_ID_1,SPACE_ID_2"
```

Либо вручную в `~/.copilot/mcp-config.json` (или `.mcp.json` в корне проекта):

```json
{
  "mcpServers": {
    "knowledgebase": {
      "type": "http",
      "url": "https://knowledgebase.dodois.io/mcp",
      "headers": {
        "Authorization": "Bearer <MCP_TOKEN>",
        "Mcp-Mode": "Write",
        "Mcp-Write-Spaces": "SPACE_ID_1,SPACE_ID_2"
      }
    }
  }
}
```

Copilot CLI, как и Claude Code, подключается к удалённому MCP по
Streamable HTTP напрямую — прокси-мост не требуется.

### VS Code (GitHub Copilot Chat)

VS Code использует отличную от других клиентов схему JSON: корневой ключ
называется `servers` (а не `mcpServers`).

Создайте файл `.vscode/mcp.json` в корне вашего проекта (или настройте
глобально через палитру команд `MCP: Open User Configuration`):

```json
{
  "servers": {
    "knowledgebase": {
      "type": "http",
      "url": "https://knowledgebase.dodois.io/mcp",
      "headers": {
        "Authorization": "Bearer <MCP_TOKEN>",
        "Mcp-Mode": "Write",
        "Mcp-Write-Spaces": "SPACE_ID_1,SPACE_ID_2"
      }
    }
  }
}
```

### Cursor

Cursor поддерживает удалённые MCP-серверы по Streamable HTTP напрямую, без
прокси-утилит (нужен Cursor v0.48.0 или новее).

Откройте Settings → Tools & Integrations → MCP Tools, добавьте новый
сервер со следующими параметрами (или пропишите в `.cursor/mcp.json`,
глобально — `~/.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "knowledgebase": {
      "url": "https://knowledgebase.dodois.io/mcp",
      "headers": {
        "Authorization": "Bearer <MCP_TOKEN>",
        "Mcp-Mode": "Write",
        "Mcp-Write-Spaces": "SPACE_ID_1,SPACE_ID_2"
      }
    }
  }
}
```

### Windsurf

Windsurf поддерживает удалённые MCP-серверы по Streamable HTTP напрямую,
через поле `serverUrl`. Конфигурация хранится глобально в
`~/.codeium/windsurf/mcp_config.json`:

```json
{
  "mcpServers": {
    "knowledgebase": {
      "serverUrl": "https://knowledgebase.dodois.io/mcp",
      "headers": {
        "Authorization": "Bearer <MCP_TOKEN>",
        "Mcp-Mode": "Write",
        "Mcp-Write-Spaces": "SPACE_ID_1,SPACE_ID_2"
      }
    }
  }
}
```

Если в вашей версии Windsurf соединение зависает — попробуйте заменить
`serverUrl` на `url`, схема периодически меняется между релизами.

### Другой MCP-клиент (общий случай)

Любой клиент, поддерживающий Streamable HTTP MCP-транспорт с кастомными
заголовками, подключается так же:

- URL: `https://<KNOWLEDGEBASE_DOMAIN>/mcp`
- Заголовок `Authorization: Bearer <MCP_TOKEN>` — обязателен
- Заголовок `Mcp-Mode: Write` — опционален, включает запись
- Заголовок `Mcp-Write-Spaces` — опционален, сужает запись до конкретных
  пространств (см. раздел 4)

Если клиент не умеет задавать кастомные HTTP-заголовки для удалённых
серверов — подключение к этому MCP-серверу для вас недоступно, обратитесь
в команду Базы Знаний.

### Если статус подключения завис на «Connecting»

В GitHub Copilot CLI это почти всегда означает не проблему транспорта, а
неверный или ещё не выданный токен: сервер отвечает `401 Unauthorized`,
после чего клиент вместо явной ошибки авторизации пытается запустить
OAuth-флоу (discovery `.well-known/oauth-authorization-server`), которого
у этого MCP-сервера нет — и зависает в состоянии `needs-auth`. Проверьте
`~/.copilot/logs/` на строки `HTTP 401` / `OAuth authentication failed` и
убедитесь, что токен реально выдан админом (раздел 1) и правильно
подставлен в заголовок `Authorization: Bearer <MCP_TOKEN>`.

## 7. Реальные права не меняются

Доступ к конкретным пространствам (можете ли вы читать/редактировать) —
тот же самый, что и везде в Базе Знаний, и не зависит от того, каким
способом вы подключились. Заголовки выше только сужают то, что и так
разрешено вашему аккаунту, но никогда не расширяют его.
