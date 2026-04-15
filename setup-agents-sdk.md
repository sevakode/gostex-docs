# Настройка Anthropic Agents SDK (OAuth через подписку)

## Зачем?
OAuth через подписку Claude Max/Pro значительно дешевле прямого API.
Использует твою месячную подписку вместо поштучной оплаты токенов.

## Шаги

1. Установи Anthropic Agents SDK CLI:
   ```bash
   npm install -g @anthropic-ai/claude-code
   ```

2. Авторизуйся через браузер:
   ```bash
   claude
   # В диалоге выбери: "Get API access from Claude.ai" → OAuth
   ```

3. Скажи OpenClaw использовать его:
   > "Настрой использование Anthropic Agents SDK через OAuth"

## OpenAI Codex OAuth (ChatGPT подписка)

1. Установи Codex CLI:
   ```bash
   npm install -g @openai/codex
   ```

2. Авторизуйся:
   ```bash
   codex
   # Выбери OAuth → войди в свой ChatGPT аккаунт
   ```

3. Скажи OpenClaw:
   > "Настрой Codex OAuth для OpenAI моделей"

## Статус
- [ ] Anthropic Agents SDK OAuth
- [ ] OpenAI Codex OAuth
