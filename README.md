# ansible-role-traefik

Ansible role to deploy [traefik(v2.x)](https://docs.traefik.io/) binary, systemd unit, and dynamic [file provider](https://docs.traefik.io/providers/file/) configs.

- [ansible-role-traefik](#ansible-role-traefik)
  - [Features](#features)
  - [Installation](#installation)
  - [How to Use](#how-to-use)
    - [Role Variables](#role-variables)
    - [Templates](#templates)
    - [Extra file provider configs](#extra-file-provider-configs)
  - [Real world usage](#real-world-usage)
    - [Prepare ansible vault](#prepare-ansible-vault)
    - [Deploy traefik with Let's Encrypt and cloudflare provider](#deploy-traefik-with-lets-encrypt-and-cloudflare-provider)
    - [Deploy traefik with user defined TLS certficates](#deploy-traefik-with-user-defined-tls-certficates)
    - [Dashboard config](#dashboard-config)
  - [Credits](#credits)
  - [License](#license)

## Features
- Install specified version/arch(e.g.: [v2.2.1_linux_amd64](https://github.com/containous/traefik/releases) traefik on target hosts, managed by systemd.
  - Only tested on Debian/Ubuntu system.
- Force https enabled by default.
- Support [User defined](https://docs.traefik.io/https/tls/#user-defined) TLS certificates.
- Support [Let's Encrypt Automatic HTTPS](https://docs.traefik.io/https/acme/#lets-encrypt) with dnsChallenge.
- Support dynamic configurations with file provider. 

## Installation

In ansible playbook project's root directory, run 

`git submodule add https://github.com/wych42/ansible-role-traefik.git roles/traefik`

The directory structure should be like: 
<details>
  <summary>example of playbook directory structure:</summary>

```
ansible-playbook
├── files
│   ├── certs
│   │   ├── example.com.cert
│   │   └── example.com.key
│   ├── server-01.example.com
│   │   └── traefik
│   │       └── conf.d
│   │           ├── middleware_basic_auth.toml
│   │           └── ...
├── group_vars
├── host_vars
├── inventory
└── roles
    ...
    ├── traefik <-- this repo
    ...
```
</details>

## How to Use

### Role Variables

Variables can be overridden by `group_vars` or `host_vars` in playbook. .e.g:

``` yml
# group_vars/all.yml
traefik_install_dir: /usr/local/traefik/bin
```

<details>
  <summary>avaliable variables:</summary>

```yml
traefik_install_dir: /usr/local/bin
traefik_bin_file: "{{ traefik_install_dir }}/traefik"

traefik_install_version: 2.2.1
traefik_install_arch: linux_amd64
traefik_binary_url: https://github.com/containous/traefik/releases/download/v{{ traefik_install_version }}/traefik_v{{ traefik_install_version }}_{{ traefik_install_arch }}.tar.gz
traefik_checksum: sha256:04139683e0cd6fc4e98eae9d469490d1a85074a0d810f640296f81991763657c

traefik_config_dir: /etc/traefik
traefik_extra_config_dir: "{{ traefik_config_dir }}/conf.d/"
traefik_config_file: "{{ traefik_config_dir }}/config.toml"
traefik_acme_file: "{{ traefik_config_dir }}/acme.json"
traefik_env_file: "{{ traefik_config_dir }}/env"
traefik_cert_dir: "{{ traefik_config_dir }}/ssl"
traefik_log_dir: "/var/log/traefik"
traefik_systemd_unit_file: /etc/systemd/system/traefik.service

traefik_config_template: traefik.toml
traefik_config_tls_template: traefik.tls.toml
traefik_systemd_unit_template: traefik.service
traefik_env_template: traefik.env

traefik_enable_docker_provider: true
traefik_enable_force_https: true
traefik_enable_acme: true
traefik_acme_ca_server: https://acme-v02.api.letsencrypt.org/directorye
traefik_san_domains: #optional
  - example.com 
```
</details>

### Templates

Templates can be overridden by custom template file with same name in playbook's `templates/` dir. .e.g: `templates/traefik.toml.j2`.

Avaliable templates are:

- `traefik.toml.j2`: template for main config.
- `traefik.service.j2`: template for systemd unit file.
- `traefik.env.j2`: template for env file, used in systemd unit, provides envs like `cloudflare_api_key`.
- `traefik.tls.toml.j2`: template for conf.d/tls.toml, used if `traefik_san_domains` is defined, works with [User defined](https://docs.traefik.io/https/tls/#user-defined) TLS certificates.


### Extra file provider configs

Put extra file provider configs for host in `files/{{ inventory_hostname }}/traefik/conf.d/`, like `middleware_basic_auth.toml` in the example above, and those files will be sync to target host's  `{{ traefik_extra_config_dir }}`, .e.g: `/etc/traefik/conf.d`.


## Real world usage

All commands should be run in the root of ansible-playbook project directory.

### Prepare ansible vault

skip if already using ansible vault.

Generate random vault password:

```bash
openssl rand -hex 32 > .vault_pass
```

### Deploy traefik with Let's Encrypt and cloudflare provider

Use ansible-vault to encrypt acme_email address, cloudflare_email and cloudflare_api_key:

```bash
ansible-vault encrypt_string --vault-password-file=./.vault_pass YOUR_ACME_EMAIL
ansible-vault encrypt_string --vault-password-file=./.vault_pass YOUR_CLOUDFLARE_EMAIL
ansible-vault encrypt_string --vault-password-file=./.vault_pass YOUR_CLOUDFLARE_API_KEY
```

Create file `group_vars/all.yml` with:
```yml
traefik_enable_acme: true
traefik_acme_email: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          63346337343737303438666530623534613636353934386630386338643230346237616338366164
          ...truncated...
traefik_acme_provider: cloudflare
traefik_envs:
  - key: CLOUDFLARE_EMAIL
    value: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          64376361653234663031613861343032376230646137396365363464353933366530383530376634
          ...truncated...
  - key: CLOUDFLARE_API_KEY
    value: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          64376361653234663031613861343032376230646137396365363464353933366530383530376634
          ...truncated...
```

Create file `files/server-01.example.com/traefik/conf.d/middleware_basic_auth.yml` with:

```toml
[http.middlewares.defaultAuth.basicAuth]
  users = [
    # foo:bar
    "foo:$apr1$qlk7za6o$T25jouGKs1WeutaH.UeSE/"
  ]
```

Create playbook `traefik.yml` with:
```yml
- hosts: "{{ host }}"
  become: true
  roles:
    - { role: "traefik", tags: "traefik" }
```

Run ansible-playbook and deploy:
```bash
ansible-playbook --vault-password-file=./.vault_pass -i inventory_file --extra-vars "host=server-01.example.com" traefik.yml
```

**Note** :Remember change `server-01.example.com` to your real hostname in inventory file; 

### Deploy traefik with user defined TLS certficates 

Assuming you alredy got wildcard certficates for domain *.example.com,and copy files:
```
files/certs/example.com.cert
files/certs/example.com.key
```

Encrypt them with ansible-vault:
```bash
ansible-vault encrypt --vault-password-file=./.vault_pass  files/certs/example.com.cert
ansible-vault encrypt --vault-password-file=./.vault_pass  files/certs/example.com.key
```

Create `group_vars/all.yml` with:
```yml
traefik_san_domains:
  - example.com
traefik_enable_acme: false
```

Create playbook `traefik.yml` with:
```yml
- hosts: "{{ host }}"
  become: true
  roles:
    - { role: "traefik", tags: "traefik" }
```

Run ansible-playbook and deploy:
```bash
ansible-playbook --vault-password-file=./.vault_pass -i inventory_file --extra-vars "host=server-01.example.com" traefik.yml
```

### Dashboard config

Enable dashboard and check health.

Create `files/server-01.example.com/traefik/conf.d/dashboard.toml`  with:
```toml
[http.routers.apiping]
  rule = "Host(`server-01.example.com`) && Path(`/ping`)"
  entrypoints = ["https"]
  service = "ping@internal"
  [http.routers.apiping.tls]

[http.routers.api]
  rule = "Host(`server-01.example.com`)"
  entrypoints = ["https"]
  service = "api@internal"
  middlewares = ["defaultAuth"]
  [http.routers.api.tls]
```

## Credits

- Rewrite from [kibatic/ansible-traefik](https://github.com/kibatic/ansible-traefik)

## License

MIT
