## Language
- Communicate with user, plans, plan-mode: English
- Tech docs (code, context, rules, CLAUDE.md, skills): English
- AI prompts: English
- User-facing docs (README): English

## Behavior

- No "Great question!", no filler, no water.
- NEVER use AskUserQuestion tool. Ask questions as plain text in chat instead.
- ALL deployments via GitHub CI/CD only. Direct server access (SSH, container restarts) only for emergency debugging of broken production.

## Task Planning
- Use TodoWrite for multi-step tasks (>1 step)
- When user asks for team/swarm of agents: use TeamCreate, not TaskCreate

## Security

- NEVER ask user to write secrets in chat
- Instead: provide instructions where to store them securely
  - Local: `.env` files, config files
  - CI/CD: GitHub Actions secrets
- ALWAYS ask before Deploy/push to main/production
- ALWAYS add secrets to `.gitignore`: `.env`, `*.key`, `credentials.json`, `secrets/`
- Be cautious with external actions (push, deploy, send messages, create PRs). Ask before acting externally when uncertain.
