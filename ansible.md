# Ansible

Use this structure for infrastructure automation with static INI inventories, local roles, and Ansible Vault. Create only the files and directories needed.

## Structure

```text
ansible-project/
├── README.md
├── ansible.cfg
├── .ansible-lint
├── .gitignore
├── requirements.txt                 # Python/pip dependencies
├── collections/requirements.yml     # Collection dependencies, if needed
├── inventories/
│   ├── staging/
│   │   ├── hosts.ini
│   │   ├── group_vars/
│   │   │   ├── all/
│   │   │   │   ├── vars.yml
│   │   │   │   └── vault.yml
│   │   │   └── web_servers.yml
│   │   └── host_vars/web01.yml
│   └── production/                  # Same layout as staging
├── playbooks/
│   ├── site.yml
│   └── web_servers.yml
├── scripts/
│   ├── encrypt.sh
│   └── decrypt.sh
└── roles/
    └── web_service/
        ├── README.md
        ├── defaults/main.yml
        ├── meta/argument_specs.yml
        ├── tasks/main.yml
        ├── handlers/main.yml
        ├── templates/service.conf.j2
        └── files/
```

## Setup

Run commands from the repository root. Use `python -m venv` and record pinned Python dependencies, including Ansible and ansible-lint, in root `requirements.txt`. Keep optional collection dependencies in `collections/requirements.yml`.

```sh
python -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements.txt
```

Configure local role discovery and the INI inventory plugin in `ansible.cfg`:

```ini
[defaults]
roles_path = ./roles

[inventory]
enable_plugins = ini
```

Document supported Python and Ansible versions and usage in `README.md`. Ignore `.venv/` and `.vault_pass` in `.gitignore`.

## Inventories and variables

- Separate inventories by environment; use `hosts.ini` and select an environment explicitly with `-i`.
- Keep YAML variable files beside the inventory under `group_vars/` and `host_vars/`. Use `.yml` for variable files, tasks, and playbooks.
- Put environment-wide values in `group_vars/all/vars.yml`, encrypted secrets in `group_vars/all/vault.yml`, group values in matching group files, and host exceptions in files matching inventory aliases.
- Use `snake_case` for group names, role names, and custom variables.
- Prefix every role-defined variable with `<role_name>_`, including defaults, internal variables, registered results, `set_fact` values, and custom loop variables. Preserve the prefix in inventory and playbook overrides; for example, `web_service_port`. Keep Ansible-provided variables and module parameter names unchanged.
- Put overridable role inputs in `defaults/main.yml`; reserve optional `vars/main.yml` for internal values.
- Document required role inputs as commented examples in `defaults/main.yml`, such as `# web_service_port: 8080`, leaving them undefined until supplied by the caller. Declare each under `argument_specs.main.options` in `meta/argument_specs.yml` with `required: true`, its type, and a description; do not add a default in the argument specification.
- Use Ansible Vault for secrets. Commit encrypted vault files; keep plaintext secrets and decryption credentials out of Git.
- Use `no_log: true` on tasks whose arguments or output expose secrets, and `diff: false` when managing sensitive file content. Never print secret variables through debug tasks, task names, or loop labels.

## Playbooks and roles

- Keep roles local and focused on one responsibility. Playbooks select hosts and compose roles; `playbooks/site.yml` composes normal configuration runs. Keep destructive or one-off operations separate.
- Start role tasks in `tasks/main.yml`; split by responsibility as needed. Keep templates and static files with their owning role; omit unused directories.
- For file content containing variables, use `.j2` templates rendered with `ansible.builtin.template`; do not embed the content inline in role tasks or generate it with `echo`, `printf`, or heredocs. Copy static files from `files/` with `ansible.builtin.copy`. This rule applies to file content; ordinary task arguments, names, and messages may use variables.
- Set explicit `owner`, `group`, and quoted `mode` values when managing files and directories. Use restrictive permissions such as `'0600'` for secret files, including `.vault_pass`.
- Name plays and tasks by intent and use fully qualified module names.
- Make roles idempotent: once the desired state is reached, rerunning with unchanged inputs and target state must cause no changes or unnecessary service restarts.
- Prefer declarative modules. Guard necessary `command` or `shell` operations with `creates`, `removes`, or explicit state checks, and use accurate `changed_when` conditions. Reporting no change does not make an operation idempotent; do not hide real changes with `changed_when: false`.
- Use handlers for service actions triggered by changes. Scope privilege escalation to the plays or tasks that need it.
- Document each role's purpose, inputs, requirements, and usage in its `README.md`.

## Vault scripts

Provide executable `scripts/encrypt.sh` and `scripts/decrypt.sh`. Each accepts exactly one existing file path and encrypts or decrypts it in place using Ansible Vault.

- Resolve repository-root `.vault_pass` relative to the script location. Use it if present; fail clearly if unreadable. Otherwise let Ansible Vault prompt with hidden input, without saving the password.
- Quote paths, resolve relative target paths from the caller's working directory, validate arguments, and propagate command failures. Never print passwords.

```sh
./scripts/encrypt.sh inventories/staging/group_vars/all/vault.yml
./scripts/decrypt.sh inventories/staging/group_vars/all/vault.yml
```

## Lint and inspect

```sh
ansible-lint
ansible-inventory -i inventories/staging/hosts.ini --graph
ansible-playbook -i inventories/staging/hosts.ini playbooks/site.yml --syntax-check
ansible-playbook -i inventories/staging/hosts.ini playbooks/site.yml --list-hosts
```
