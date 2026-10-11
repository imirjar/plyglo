# Microservices Courses Platform With Traefik and Authelia
```text
██████╗  ██╗    ██╗   ██╗  ██████╗  ██╗       ██████╗
██╔══██╗ ██║    ╚██╗ ██╔╝ ██╔════╝  ██║      ██╔═══██╗
██████╔╝ ██║     ╚████╔╝  ██║  ███╗ ██║      ██║   ██║
██╔═══╝  ██║      ╚██╔╝   ██║   ██║ ██║      ██║   ██║
██║      ███████╗  ██║    ╚██████╔╝ ███████╗ ╚██████╔╝
╚═╝      ╚══════╝  ╚═╝     ╚═════╝  ╚══════╝  ╚═════╝

                 .md  ⇄  A  ⇄  文

                    plyglo.com
```
              
## TRAEFIK TLS
### Add acme directory and 
```
touch traefik/certs/acme.json traefik/acme-staging.json
chmod 600 traefik/certs/acme.json traefik/certs/acme-staging.json
```

# Смените владельца на UID/GID, под которым работает пользователь в контейнере Authentik.
# В официальном образе это пользователь с UID 1000 (обычно).
sudo chown -R 1000:1000 auth/data auth/certs

# Дайте права на запись
sudo chmod -R 775 auth/data auth/certs