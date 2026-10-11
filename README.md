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
              
## Prepare project
### Add acme files and grant 600
```
touch traefik/certs/acme.json traefik/certs/acme-staging.json
chmod 600 traefik/certs/acme.json traefik/certs/acme-staging.json
```
### Change owner for authentic files
```
sudo chown -R 1000:1000 ./auth/media ./auth/certs ./auth/data
sudo chmod -R 775 ./auth/media ./auth/certs ./auth/data
```

## Start project
```
docker compose up --build -d
```
