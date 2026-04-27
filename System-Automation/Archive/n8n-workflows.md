---
description: "n8n workflow creation, editing, and debugging rules"
globs:
  - "n8n_setup.md"
  - "**/*n8n*"
  - "**/*workflow*"
---

# n8n Workflow Rules

## Infrastructure

- n8n host: https://n8n.srv1016866.hstgr.cloud/
- User WhatsApp: 352621486096
- WhatsApp Phone Number ID: 1020770907789933

## Credentials

| Name | Type | ID |
|------|------|----|
| OpenAi account | openAiApi | f4CHgV75xGzGHl4B |
| Supabase-Second Brain | supabaseApi | Haz6HNm3WAjfQrYa |
| Send WhatsApp Personal | whatsAppApi | kOBiI6xLEZFkr28v |
| n8n API | n8nApi | irS5M5SchH4BxMdB |

## Critical Node Rules

1. **OpenAI → MUST use native node** (`@n8n/n8n-nodes-langchain.openAi`, typeVersion 1.8)
   - Credential is configured to BLOCK use inside HTTP Request nodes
   - Response at: `$input.first().json.message.content`

2. **Supabase → MUST use HTTP Request** with `predefinedCredentialType: supabaseApi`
   - Native Supabase node introspects pgmq schema → permission errors
   - Always set `neverError: true` + pair with a Check Write Code node

3. **IF nodes → use `typeValidation: "loose"`** not "strict"
   - Use integer flags (1/0) not booleans for branch conditions

4. **No `$vars.*`** — free plan has no Variables feature. Hardcode in Code nodes.

5. **No cross-branch node references** — never call `$('NodeName')` if that node may not have run.
   Only use `$input.first()` or guaranteed ancestors.

6. **Always validate writes** — Check Write Code node reads from Supabase response body,
   throws if statusCode ≠ 200/201, then emits clean {slug, title, project, topic}.

7. **Execution ID offset** — Test Webhook gets ID N, triggered workflow gets ID N+1.

8. When creating workflows via API, always set `availableInMCP: true` in settings.

## Key Workflow IDs

- Phone capture: `zwAM6zuOkwFsXz60`
- Update Workflow: `IHhN14A7NaCHeocZ`
- Test Webhook: `w4mhvZqpQwt8yyAL`
- Supabase Query: `cqS9UPhw014jAPWS`
