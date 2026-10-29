# Infraestrutura do Durval AI: LiteLLM (Ansible)

Playbooks para instalar no Ubuntu 22.04+ o [LiteLLM](https://docs.litellm.ai/) standalone
(pip + systemd, sem Docker) e publicá-lo pelo nginx da VM na porta 443:

| Endpoint (HTTPS 443) | Destino |
|---|---|
| `https://litellm.<ip>.nip.io/mcp` | MCP servers, pelo gateway MCP do LiteLLM (`/litellm/mcp/`) |
| `https://litellm.<ip>.nip.io/litellm/ui` | Console admin do LiteLLM |
| `https://litellm.<ip>.nip.io/litellm/...` | API do LiteLLM (usada pelo console; também atende `/litellm/v1/...`) |

O hostname é montado do IP de cada host (`ansible_host`, com `.` trocado por `-`):
`litellm.203-0-113-10.nip.io` em dev e `litellm.203-0-113-20.nip.io` em qas. Para usar
outro, defina `dominio_litellm` no host do inventário.

| Playbook | Componente | Porta |
|---|---|---|
| `playbooks/postgresql.yml` | Usuário e banco `litellm` no PostgreSQL compartilhado (só conexões locais) | 5432 |
| `playbooks/litellm.yml` | LiteLLM 1.105.7 em `/var/lib/litellm/venv`, serviço `litellm` | 127.0.0.1:4000 |
| `playbooks/nginx.yml` | Site `durvalai.conf` no nginx (TLS, endpoints MCP e console) | 80/443 |
| `playbooks/site.yml` | Os três, na ordem correta | |

O PostgreSQL 18 e o nginx não são instalados aqui: vêm da solução
`01-cairbar-identity-provider` (`playbooks/postgresql.yml` e `playbooks/keycloak.yml`),
que precisa rodar antes. Os playbooks daqui falham se não os encontrarem.

Versões, portas, modelos e MCP servers ficam em `global_vars/main.yml`.

## Como funciona

- O LiteLLM roda com `SERVER_ROOT_PATH=/litellm`: todas as rotas dele ficam sob esse
  prefixo, que o nginx repassa sem alteração. O console fica em `/litellm/ui`.
- `/mcp` é reescrito para `/litellm/mcp/`, o gateway MCP do LiteLLM, que agrega os MCP
  servers cadastrados em `litellm_mcp_servers` (ou pelo console). A autenticação é por
  chave virtual do LiteLLM (`Authorization: Bearer sk-...`), criada no console.
- `durvalai_admin_allowed_cidrs` restringe por IP o acesso a `/litellm/` (console e API);
  vazio, fica aberto e protegido só pelo login. Não afeta `/mcp`.
- O console exige banco: chaves virtuais, usuários, gastos e modelos/MCP servers
  cadastrados pela UI ficam no banco `litellm`.

## Cadastrar um MCP server

Cada MCP server a ser criado roda na VM (ex.: em `127.0.0.1:8100`, transporte
streamable HTTP) e entra em `litellm_mcp_servers`:

```yaml
litellm_mcp_servers:
  durval_crm:
    url: "http://127.0.0.1:8100/mcp"
    transport: http
    description: "Consultas ao DurvalCRM"
```

Depois rode `playbooks/litellm.yml`. Os clientes MCP usam
`https://litellm.<ip>.nip.io/mcp` com uma chave virtual.

## Preparação (uma vez)

Execute a partir da pasta `ansible/`:

```bash
ansible-galaxy collection install -r requirements.yml -p collections
```

```bash
cp global_vars/vault.yml.example global_vars/vault.yml
```

Defina os segredos em `global_vars/vault.yml` e criptografe:

```bash
ansible-vault encrypt global_vars/vault.yml --vault-password-file .vault_pass
```

```bash
podman machine start
```

## Execução

No macOS, rode sem `SSH_AUTH_SOCK`: o socket do ssh-agent não existe dentro da VM do
podman e o container não sobe (a autenticação usa o arquivo `.pem`).

```bash
env -u SSH_AUTH_SOCK ansible-navigator run playbooks/site.yml --vault-password-file .vault_pass
```

Apenas um componente (ou um host com `--limit vm-exp-cairbar-360`):

```bash
env -u SSH_AUTH_SOCK ansible-navigator run playbooks/litellm.yml --vault-password-file .vault_pass
```

Sem container (usa o Ansible local):

```bash
ansible-navigator run playbooks/site.yml --ee false --vault-password-file .vault_pass
```

## Operação

Na VM: `systemctl status litellm`, `journalctl -u litellm -f` e
`/var/log/nginx/durvalai-access.log`. Para trocar de versão, altere `litellm_version`
e rode `playbooks/litellm.yml`; as migrações do banco são aplicadas na partida.
