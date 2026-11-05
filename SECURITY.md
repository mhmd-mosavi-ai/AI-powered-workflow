# Security

- Never commit API keys, access tokens, passwords, cookies, private keys, or exported credential data.
- Configure integrations through n8n Credentials.
- Keep customer data, execution logs, and production exports out of this repository.
- If a secret is exposed, revoke/rotate it immediately. Removing it from the latest commit does not remove it from Git history.
- Review every workflow export before publishing it.
