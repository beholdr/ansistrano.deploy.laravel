# Ansistrano Deploy Laravel

Ansible role for simple deployment of a Laravel app with [ansistrano.deploy](https://galaxy.ansible.com/ui/standalone/roles/ansistrano/deploy/) and docker compose.

## Install

Ansistrano role uses [ansible.posix](https://galaxy.ansible.com/ui/repo/published/ansible/posix/) collection, so you need to install it too. You can use `requirements.yml` file:

```yml
---
roles:
  - name: ansistrano.deploy.laravel
    src: https://github.com/beholdr/ansistrano.deploy.laravel

collections:
  - name: ansible.posix
    version: 1.5.4
```

## Usage

Example `inventory.yml`:

```yml
---
servers:
  hosts:
    dev:
      ansible_user: root
      ansible_host: dev.sitename.com # or IP address
      app_name: sitename-dev
      app_url: dev.sitename.com
      app_path: /srv/sitename-dev
      app_environment: dev
    production:
      ...
```

Example `playbook.yml`:

```yml
---
- name: Deploy
  hosts: servers
  gather_facts: false

  vars:
    ansistrano_deploy_from: "{{ playbook_dir }}/../../"
    ansistrano_deploy_to: "{{ app_path }}"
    ansistrano_keep_releases: 5

    # you can override some variables
    app_shared_paths:
      - "storage"
      - "your/other/path"

  tasks:
    - name: Laravel deploy role
      ansible.builtin.include_role:
        name: ansistrano.deploy.laravel
```

### Env and compose files

You can put `.env` and `compose.yml` files for each of your environments in the project root:

```
├─ .env.dev
├─ .env.production
├─ compose.production.yml
└─ compose.dev.yml
```

Files could be plain or encrypted with ansible vault. Corresponding file (determined by `app_environment`) will be uploaded to your server and decrypted automatically.

Also you can use inline encrypted variables in your `.env` files like this:

```sh
APP_SECRET={{ vault_app_secret }}
APP_SECRET_DEV={{ vault_app_secret_dev }}
```

Then add inline encrypted secret vars in your inventory:

```yml
---
servers:
  hosts:
    dev:
      ansible_user: root
      ansible_host: dev.sitename.com # or IP address
      app_name: sitename-dev
      app_url: dev.sitename.com
      app_path: /srv/sitename-dev
      app_environment: dev
      # encrypted var for the current environment
      vault_app_secret_dev: !vault |
                            $ANSIBLE_VAULT;1.2;AES256;dev
                            3735363836303930623263613...3864
```

## Variables

The `defaults` role vars:

- `app_environment`: environment (default: `dev`)
- `app_container_name`: app container name (default: `php`)
- `app_http_user`: user name for files ownership (default `www-data`)
- `app_root_user`: user name for root (default `root`)
- `app_exclude_paths`: list of paths to exclude by rsync
- `app_shared_paths`: list of paths to use as shared between releases
- `app_writable_paths`: list of paths to create and set writable
- `app_docker_commands`: list of commands to run in the docker container on deployment

You can override these vars or, if you just want to add some items to lists without overriding defaults, you can use `_add` variables:

- `app_exclude_paths_add`
- `app_shared_paths_add`
- `app_writable_paths_add`
- `app_docker_commands_add`

> Please note that paths in the `app_exclude_paths_add` list should start with `/` if you want to target only top level items. For example if you add to the list `vendor` it will exclude not only project-root `vendor` but any `vendor` deep in the hierarchy. If you want to exclude only project-root `vendor` use `/vendor`.

### Artisan commands with conditions

Every artisan command can include `enabled` field to determine if it should run.
For example, you can enable DB seeding on the non-production environment:

```yml
- command: php artisan migrate:fresh --seed --force
  enabled: "{{ app_environment == 'dev' }}"

- command: php artisan migrate --force
  enabled: "{{ app_environment == 'production' }}"
```

### Run command as root

By default commands run as `app_http_user`. But some commands (e.g. `composer install`) may require root privileges. You can add `root: true` flag in this case.

### Default commands

```yml
- command: composer install --prefer-dist --no-progress --no-interaction --optimize-autoloader --no-dev
  root: true
- command: composer dump-autoload --classmap-authoritative
  root: true
- command: php artisan storage:link
  root: true
- command: php artisan optimize
- command: php artisan migrate --force
```
