# yixin-api

Codex skill for using the Yixin OpenAPI portal and public APIs.

## Install

### Using `npx skills add`

Install globally for Codex:

```bash
npx skills add billionsintelligence/yixin-api --skill yixin-api -g -a codex -y
```

Install into the current project instead of globally:

```bash
npx skills add billionsintelligence/yixin-api --skill yixin-api -a codex -y
```

If the shorthand repository form does not work in your environment, use the full GitHub URL:

```bash
npx skills add https://github.com/billionsintelligence/yixin-api --skill yixin-api -g -a codex -y
```

### Using Codex Skill Installer

In Codex, ask:

```text
$skill-installer install billionsintelligence/yixin-api path skills/yixin-api
```

After installation, restart Codex so it can discover the new skill.

### Manual Install

Clone this repository and copy the skill directory into Codex's skills directory:

```bash
git clone https://github.com/billionsintelligence/yixin-api.git
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
cp -R yixin-api/skills/yixin-api "${CODEX_HOME:-$HOME/.codex}/skills/yixin-api"
```

Then restart Codex.

### Local Development Install

For local development, symlink the skill directory so edits are picked up without copying:

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
ln -s "$PWD/skills/yixin-api" "${CODEX_HOME:-$HOME/.codex}/skills/yixin-api"
```

Restart Codex after creating the symlink.
