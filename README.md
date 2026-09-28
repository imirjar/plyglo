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
              
## Prepare for local development

### 1. Configure /etc/hosts for local dev
```
127.0.0.1       plyglo.com
127.0.0.1       study.plyglo.com
127.0.0.1       app.plyglo.com
127.0.0.1       api.plyglo.com
127.0.0.1       auth.plyglo.com
127.0.0.1       localhost
255.255.255.255 broadcasthost
::1             localhost
```

### 2. Make certs in ./certs using [mkcert](https://github.com/filosottile/mkcert) for local dev
```
mkcert \
  -cert-file certs/plyglo.com.pem \
  -key-file certs/plyglo.com-key.pem \
  plyglo.com "*.plyglo.com" localhost 127.0.0.1 ::1
```

