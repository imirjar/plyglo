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
              
## hide acme 
sudo chmod 600 traefik/acme.json traefik/acme-staging.json

# Смените владельца на UID/GID, под которым работает пользователь в контейнере Authentik.
# В официальном образе это пользователь с UID 1000 (обычно).
sudo chown -R 1000:1000 auth/data auth/certs

# Дайте права на запись
sudo chmod -R 775 auth/data auth/certs