# /double — Switch Claude Account

**Переключение между двумя Claude аккаунтами.**

## Триггеры

- `/double` — переключить на другой аккаунт
- `/double primary` — переключиться на основной
- `/double fallback` — переключиться на запасной
- "switch claude" / "переключи claude"

## Использование

```
/double         # показать текущий аккаунт
/double primary # переключиться на основной (claude-cli)
/double fallback# переключиться на запасной
```

## Требования

1. Два OAuth профиля в `auth-profiles.json`:
   - `anthropic:claude-cli` — основной
   - `anthropic:fallback` — запасной

2. Скрипт переключения:
   ```bash
   cp ~/skills/public/double-claude/scripts/switch.sh.sample ~/claude-switch.sh
   chmod +x ~/claude-switch.sh
   ```

## Как это работает

1. Скрипт меняет `lastGood.anthropic` в `auth-profiles.json`
2. Перезапускает gateway
3. Следующая сессия использует новый аккаунт

## Проверка

```bash
# Текущий аккаунт
jq -r '.lastGood.anthropic' ~/.openclaw/agents/main/agent/auth-profiles.json

# Список профилей
jq -r '.profiles | keys[] | select(startswith("anthropic:"))' ~/.openclaw/agents/main/agent/auth-profiles.json
```

## Пример

```
You: /double fallback
Claw: 🔄 Переключаю на fallback...
     ✓ Обновлено
     ✓ Gateway перезапущен
     ✅ Готово! Теперь работаю с anthropic:fallback
```
