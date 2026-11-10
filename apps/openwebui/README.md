# OpenWebUI

## Manual setup after deploy

Flux deploys OpenWebUI itself. These settings live in its database and must be set again by hand (Admin Settings) on a fresh install:

| Setting | Value |
|---|---|
| Connections → OpenAI API | `https://nai.yolocal.cloud/enterpriseai/gateway/v1`, bearer auth, NAI API key |
| Interface → Task model | `gemma-12b-taskmodel` (the NAI endpoint name) |
| External Tools → MCP server `ntnx-mcp` | `https://nai.yolocal.cloud/enterpriseai/gateway/mcp/nutanix-mcp-new`, bearer auth (NAI API key) |
| External Tools → MCP server `zy-gpu-k8s-mcp` | `https://nai.yolocal.cloud/enterpriseai/gateway/mcp/k8s-mcp-new`, bearer auth (NAI API key) |

The MCP servers have to be registered in NAI first. Then add the skills below.

## Skills (manual)

OpenWebUI stores skills in its database (`webui.db`, the `skill` table), not in files. Flux does **not** deploy them: a fresh OpenWebUI starts with no skills, and they must be added by hand.

[skills/](skills/) is the source of truth, one folder per skill:

```
skills/<id>/SKILL.md        # the folder name is the skill id
```

### Adding skills to a new OpenWebUI

After Flux deploys OpenWebUI and you've signed up (the first account becomes admin), for each `skills/<id>/SKILL.md`:

1. Go to **Workspace → Skills** and create a new skill.
2. Set the id to the folder name (lowercase letters, digits, `-` or `_`).
3. Set the name. Existing skills use the id in title case, e.g. `nutanix-vm-list` → `Nutanix Vm List`.
4. Copy the `description` from the SKILL.md frontmatter, and paste the whole file as the content.

### Keeping git in sync

When you change a skill in the UI, copy it back into `skills/<id>/SKILL.md` and commit it. To export every skill's current content from a running instance:

```
kubectl -n openwebui exec deploy/openwebui-deployment -- python3 -c '
import sqlite3, json
c = sqlite3.connect("file:/app/backend/data/webui.db?mode=ro", uri=True)
print(json.dumps(dict(c.execute("select id, content from skill").fetchall())))' > skills.json
```

Then write each `content` to `skills/<id>/SKILL.md`.
