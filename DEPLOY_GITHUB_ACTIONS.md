# Deploy de jsreport con GitHub Actions

**Dominio de producción:** [reports.g360co.com](https://reports.g360co.com)

El workflow en `.github/workflows/deploy.yml` sube el código por SSH y reinicia la app con **PM2**. En producción se expone **HTTPS con Nginx** y **Let's Encrypt**; jsreport queda detrás del proxy en `127.0.0.1:5488` (puerto en `jsreport.config.json`).

## 1) Environment y secrets en GitHub

El job del workflow usa el **environment** `REPORTS_SERVER` (definido en `Settings → Environments`).

En ese environment debes crear estos **Environment secrets**: `Settings → Environments` → **REPORTS_SERVER** → *Environment secrets*:

| Secret | Descripción |
|--------|-------------|
| `SSH_HOST` | IP o dominio del servidor SSH |
| `SSH_PORT` | Puerto SSH (normalmente `22`) |
| `SSH_USER` | Usuario SSH |
| `SSH_PRIVATE_KEY` | Llave privada en formato PEM |
| `APP_DIR` | Ruta del proyecto en el servidor, p. ej. `/var/www/jsreport` |

Si el environment `REPORTS_SERVER` tiene **reglas de protección** (revisores, timer), el deploy manual puede quedar pendiente de aprobación; es esperado.

Si aparece **`Error: missing server host`**, revisa que **`SSH_HOST`** exista en el environment `REPORTS_SERVER` y no esté vacío.

## 2) DNS y firewall

### Dominio `reports.g360co.com`

1. En el DNS de **g360co.com**, crea un registro **A** (y **AAAA** si usas IPv6) para el host **`reports`** apuntando a la **IP pública** del servidor Ubuntu.
2. Espera la propagación (minutos u horas según el proveedor).
3. Comprueba:

```bash
dig +short reports.g360co.com
ping -c 2 reports.g360co.com
```

### Firewall (UFW, si lo usas)

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

No abras el puerto `5488` al público si Nginx hace de proxy en 80/443; jsreport puede quedar solo accesible en `127.0.0.1:5488`.

## 3) Dependencias en el servidor (una sola vez)

- **Node.js 20 LTS** (alineado con GitHub Actions y con `jsreport` fijado en **4.10.1**; versiones más nuevas de jsreport 4.x pueden exigir Node 22). Comprueba con `node -v`.
- `npm` (viene con Node).
- PM2 global: `sudo npm install -g pm2`
- Nginx y Certbot:

```bash
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
```

## 4) Clonar o preparar `APP_DIR`

El workflow copia el repo a `APP_DIR`. La primera vez el job crea la carpeta con `mkdir -p`. Tras el primer deploy tendrás el código y `ecosystem.config.cjs`.

El usuario de deploy (`SSH_USER`) debe poder escribir en `APP_DIR` y ejecutar `pm2`.

PM2 al reiniciar el servidor:

```bash
pm2 startup
# Ejecuta el comando que PM2 muestre (con sudo)
pm2 save
```

## 5) Nginx: sitio y proxy a jsreport

El archivo `deploy/nginx-jsreport.conf.example` está pensado para copiarlo **tal cual** la primera vez: **solo puerto 80** y `proxy_pass` a jsreport. Así `nginx -t` no falla por certificados que aún no existen.

1. Copia el ejemplo a `/etc/nginx/sites-available/reports.g360co.com`.
2. Enlaza el sitio y recarga Nginx:

```bash
sudo ln -sf /etc/nginx/sites-available/reports.g360co.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

3. Con jsreport en PM2 (`pm2 status`), verifica en el servidor:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:5488/
curl -sS -o /dev/null -w "%{http_code}\n" -H "Host: reports.g360co.com" http://127.0.0.1/
```

## 6) Let's Encrypt (certificado TLS)

**Cuándo se solicita el certificado:** lo pides tú **una vez** (o certbot en modo interactivo), cuando el dominio ya resuelve a tu servidor y Nginx sirve el `server_name` en el puerto 80. No hace falta crear antes los `.pem` ni escribir a mano el bloque `listen 443` en el repo.

Con plugin Nginx instalado (`python3-certbot-nginx`):

```bash
sudo certbot --nginx -d reports.g360co.com
```

Certbot valida por HTTP, **crea** los ficheros en `/etc/letsencrypt/live/reports.g360co.com/` y **modifica** el archivo del sitio en `sites-available` (añade SSL y, en general, redirección 80→443). Después de eso, revisa que el bloque `:443` siga teniendo el mismo `location /` con `proxy_pass` a `5488`; si certbot dejó un `443` mínimo, copia ahí las mismas directivas `proxy_*` que en el ejemplo.

**Renovación automática:** en Ubuntu viene programado `certbot renew` (timer/cron). Comprueba con:

```bash
sudo certbot renew --dry-run
```

## 7) URL pública del API jsreport

| Uso | URL |
|-----|-----|
| Base (Studio, navegador) | `https://reports.g360co.com` |
| Render API (típico) | `https://reports.g360co.com/api/report` |

En clientes conviene usar la base `https://reports.g360co.com` sin puerto `5488`.

## 8) Ejecutar deploy desde GitHub

El deploy **solo se ejecuta a mano**: `Actions → Deploy jsreport → Run workflow` (elige rama si GitHub lo ofrece, p. ej. `main`) y confirma.

El job instala **npm 11** en el runner (Node 20 trae npm 10; el `package-lock.json` debe validarse con la misma familia de npm que usaste al generarlo, si no `npm ci` falla con *out of sync*). Luego `npm ci`, sube archivos a `APP_DIR`, en el servidor ejecuta **`npx npm@11 ci --omit=dev`** (sin depender de una instalación global de npm 11) y **`pm2 reload`** o **`pm2 start`**.

## 9) Notas importantes

- **npm y el lockfile**: si al hacer `npm ci` local o en CI ves *package.json and package-lock.json are not in sync*, ejecuta `npm install` en tu máquina, commitea el `package-lock.json` y mantén en CI/servidor el uso de **npm 11** como en el workflow, o regenera el lock solo con **npm 10** y no mezcles criterios.
- **fs-store**: `data/` es estado persistente; el workflow usa `rm: true` y **reemplaza** `APP_DIR`. Haz backup de `data/` o ajusta la estrategia de despliegue si no quieres perder plantillas entre deploys.
- **Autenticación**: revisa `jsreport.config.json` antes de exponer el sitio (usuario/contraseña, `authentication.enabled`).
- **Alternativa sin PM2**: `deploy/jsreport.service.example` con systemd y cambiar el último paso del workflow a `systemctl restart jsreport`.
