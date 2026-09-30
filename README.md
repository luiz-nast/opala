<div align="center">

# 🚗 opala

**A base do meu servidor: um VS Code no navegador que controla o Docker do host.**

![Alpine](https://img.shields.io/badge/Alpine-3.22-0D597F?logo=alpinelinux&logoColor=white)
![code-server](https://img.shields.io/badge/code--server-latest-007ACC?logo=visualstudiocode&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-compose-2496ED?logo=docker&logoColor=white)
![Caddy](https://img.shields.io/badge/Caddy-HTTPS_automático-1F88C0?logo=caddy&logoColor=white)

</div>

---

## ✨ O que é

O `opala` é o container de **code-server** (VS Code web) de onde eu edito e subo todos os outros projetos do servidor. Ele recebe o socket do Docker do host, então um `docker compose up` rodado lá dentro sobe containers **no host**.

```
                    internet
                       │
              ┌────────▼────────┐
              │  Caddy (host)   │  HTTPS automático, :80/:443
              └───┬────┬────┬───┘
                  │    │    │   reverse_proxy só para 127.0.0.1
      ┌───────────┘    │    └──────────────┐
      ▼                ▼                   ▼
 opala :8080     projeto A :808x     projeto B :808x ...
 (code-server)
      │  /var/run/docker.sock
      └──────────► Docker do host
```

## 🖥️ Servidor

| | |
|---|---|
| SO | Alpine Linux 3.22 (sem bash, shell é `sh`/ash) |
| Máquina | 1 vCPU / 4 GB RAM (Hostinger) |
| Pacotes do host | `docker`, `docker-cli-compose`, `caddy`, `git`, `github-cli`, `curl` |
| Serviços (OpenRC) | `docker` e `caddy` iniciando no boot |

## 📁 Estrutura

```
/root/opala/
├── compose.yaml            # o serviço code-server
├── code_server.dockerfile  # code-server + docker CLI dentro
├── .env                    # PASSWORD do code-server (fora do git; ver .env.example)
├── config/                 # ~/.config do code-server (só tem o config.yaml com senha: fora do git)
├── data/                   # estado do code-server; no git só vão settings.json e as listas de extensões
└── project/                # um diretório por projeto (fora deste repo)
```

Montagens do container:

| Host | Container | Por quê |
|---|---|---|
| `./project` | `/root/project` | onde ficam os projetos |
| `./config` | `/root/.config` | configuração do code-server |
| `./data` | `/root/.local/share/code-server` | extensões e estado |
| `/var/run/docker.sock` | igual | controlar o Docker do host |
| `/etc/caddy/Caddyfile` | `/root/Caddyfile` | editar o proxy de dentro do editor |

## 🌐 Portas e domínios

Cada projeto publica **só** em `127.0.0.1`, e o Caddy do host faz o HTTPS.

| Porta local | Domínio | Projeto |
|---|---|---|
| 8080 | `srv1556653.hstgr.cloud` | opala (este repo) |
| 8081 | `hellcat-redeye.premiumlts.com.br` | `project/hellcat` |
| 8082 | `hellcat-demon.premiumlts.com.br` | `project/hellcat-demon` |
| 8083 | `supra.premiumlts.com.br` | `project/supra` → [luiz-nast/supra](https://github.com/luiz-nast/supra) |
| 8090 | (sem domínio, só local) | `project/qwen` (llama.cpp) |

Próxima porta livre para projeto novo: **8084**.

`/etc/caddy/Caddyfile` completo:

```caddy
srv1556653.hstgr.cloud {
reverse_proxy 127.0.0.1:8080
}

hellcat-redeye.premiumlts.com.br {
reverse_proxy 127.0.0.1:8081
}

hellcat-demon.premiumlts.com.br {
reverse_proxy 127.0.0.1:8082
}

supra.premiumlts.com.br {
reverse_proxy 127.0.0.1:8083
}
```

DNS: um registro **A** por subdomínio, todos apontando para o IP do servidor.

## 🚀 Montar um servidor novo do zero

**1. Pacotes e serviços (como root, no Alpine):**

```sh
apk update
apk add docker docker-cli-compose caddy git github-cli curl
rc-update add docker boot
rc-update add caddy default
service docker start
```

**2. Este repo:**

```sh
git clone https://github.com/luiz-nast/opala.git /root/opala
cd /root/opala
cp .env.example .env     # defina a senha do code-server
mkdir -p project config
docker compose up -d --build
```

As extensões listadas em `data/extensions/extensions.json` precisam ser reinstaladas pelo próprio code-server (o arquivo diz quais eram).

**3. Caddy:** copie o Caddyfile acima para `/etc/caddy/Caddyfile`, depois:

```sh
caddy validate --config /etc/caddy/Caddyfile
service caddy start
```

**4. Projetos:** clone cada um dentro de `project/` e siga o README dele. Exemplo:

```sh
git clone https://github.com/luiz-nast/supra.git /root/opala/project/supra
```

**5. DNS:** aponte os subdomínios para o IP novo.

### Checklist pós-deploy

- [ ] `docker ps` mostra `opala-service1-1` em `127.0.0.1:8080`
- [ ] `https://srv1556653.hstgr.cloud` (ou o hostname novo) abre o code-server e pede senha
- [ ] dentro do code-server, `docker ps` no terminal lista os containers do host
- [ ] cada domínio da tabela responde com HTTPS válido

## 🔧 Manutenção

| Situação | Comando (em `/root/opala`) |
|---|---|
| mudou o `.env` | `docker compose up -d` (**não** use `restart`: ele não relê o `.env`) |
| atualizar o code-server | `docker compose build --pull && docker compose up -d` |
| mudou o Caddyfile | `caddy reload --config /etc/caddy/Caddyfile` |

> ⚠️ O container roda como root e tem o socket do Docker, o que na prática é root no host. Por isso ele só escuta em `127.0.0.1` e fica atrás do Caddy com senha.
