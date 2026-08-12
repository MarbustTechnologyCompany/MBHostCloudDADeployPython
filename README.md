# MBHostCloudDADeployPython

**Despliega tu propia app Python (FastAPI, Flask o Django) en tu hosting de MBHostCloud — con control total y logs en vivo.**

> ⚠️ **Guía exclusiva para hostings de MBHostCloud®.** Todos los comandos, rutas, versiones y configuraciones fueron **verificados sobre la infraestructura de MBHostCloud** (DirectAdmin + Apache). Están pensados **únicamente** para cuentas de hosting de MBHostCloud®; en otros proveedores es muy probable que no apliquen o se comporten distinto. Si tienes tu hosting con nosotros, funcionan tal cual. 💙

Esta guía es para clientes de MBHostCloud (hosting **DirectAdmin + Apache**) que quieren correr su propia aplicación **Python** (una API con FastAPI, Flask o Django, un backend, un bot, lo que sea) directamente en su cuenta, sin depender de nadie. Vas a poder subir tu código, armar tu entorno, mantener el proceso vivo aunque el servidor se reinicie, ver los logs en tiempo real, y enlazar tu dominio a la app.

> 🎁 **La mejor parte:** a diferencia de Node, en Python **NO tocas tu código.** `gunicorn` publica tu app en un socket por fuera — tu `main.py` / `app.py` / proyecto Django queda **igual** que en tu PC.

Todo lo que está aquí fue **probado en el servidor real**. Los comandos funcionan tal cual.

---

## Requisitos

- **Acceso SSH por el Terminal del panel DirectAdmin.** No necesitas root ni una llave especial: entra a tu panel → busca **"Terminal"** (o "SSH Terminal") y tendrás una consola dentro de tu cuenta. Todo se hace desde ahí.
- **Saber comandos básicos** de Linux (`cd`, `ls`, `git`, editar un archivo).
- **Tu app vive FUERA de `public_html`.** Regla de oro: `public_html` es para archivos públicos estáticos. Tu app Python va en tu HOME, por ejemplo `~/miapi`. Apache reenvía el tráfico a tu app; nadie navega los archivos directamente.

> Tu app corre como un proceso tuyo que escucha en un **socket** (un archivo dentro de tu carpeta, **sin abrir ningún puerto**). Apache reenvía `tudominio.com` a ese socket. Nunca expongas la carpeta de tu código en la web.

---

## Paso 1 — Subir tu app (fuera de `public_html`)

Entra al Terminal del panel y ponte en tu HOME. Sube el código con `git clone` (recomendado) o por FTP/File Manager a una carpeta como `~/miapi`.

```bash
cd ~                                  # tu HOME, NO public_html
git clone https://github.com/tu-usuario/tu-repo.git miapi
cd miapi
ls
```

Si prefieres subir por FTP o por el File Manager del panel, crea la carpeta `miapi` **al mismo nivel** que `public_html`, no dentro de ella.

Tu estructura debe verse así:

```
/home/tu-usuario/
├── public_html/      <- archivos públicos (NO tu app)
├── miapi/            <- aquí tu app Python
│   ├── requirements.txt
│   ├── main.py       (FastAPI)  /  app.py (Flask)  /  manage.py (Django)
│   └── ...
└── .bashrc
```

> **`requirements.txt` es clave:** lista ahí tus dependencias (`fastapi`, `flask`, `django`, etc.). Si no lo tienes, créalo con `pip freeze > requirements.txt` desde tu PC.

---

## Paso 2 — Entorno virtual (venv) + dependencias

**En MBHostCloud ya tienes Python 3 y PM2 listos por defecto — no instalas nada de eso.** Al abrir el Terminal ya están en tu PATH:

```bash
python3 --version    # Python 3.9.x  (ya disponible)
pm2 -v               # ya disponible
```

Cada app Python vive en su **entorno virtual** propio (`.venv`), aislado del resto. Créalo e instala tus dependencias:

```bash
cd ~/miapi
python3 -m venv .venv                      # crea el entorno virtual (carpeta .venv)
./.venv/bin/pip install -r requirements.txt # instala TUS dependencias
```

Además del framework, necesitas el servidor de producción **gunicorn** (y **uvicorn** si usas FastAPI/Starlette). Puedes añadirlos a tu `requirements.txt` o instalarlos directo:

```bash
# FastAPI / Starlette (ASGI):
./.venv/bin/pip install gunicorn uvicorn
# Flask / Django (WSGI):
./.venv/bin/pip install gunicorn
```

> Al **enlazar** tu app, MBHostCloud se asegura de que gunicorn (y uvicorn si aplica) estén instalados — pero tenerlos desde ya te deja probar en tu propio Terminal.

---

## ⭐ La gran ventaja: NO tocas tu código

Aquí Python te lo pone **más fácil que Node**. No modificas ni una línea de tu app.

**El problema del puerto:** si tu app escucha en un puerto (ej. `8000`), ese puerto es **del servidor entero**. Si otro cliente también usa el 8000, uno de los dos no arranca. Y encima queda un puerto abierto en el VPS.

**La solución:** corres tu app con **gunicorn**, que la publica en un **socket Unix** — un archivo dentro de TU carpeta. Como es una ruta tuya, **nunca choca con nadie** y **no abre ningún puerto**.

Lo mejor: **gunicorn liga ese socket por fuera** con la opción `-b unix:...`. Tú **no cambias tu `main.py`/`app.py`** para "escuchar en el socket" como en Node — gunicorn lo hace por ti. El mismo código sirve igual en tu PC (`uvicorn main:app` / `flask run` / `python manage.py runserver`) y en el hosting (gunicorn + socket).

Lo único que necesitas saber de tu app es su **entry point** (`modulo:objeto`):

| Framework | Servidor (gunicorn) | entry (`modulo:objeto`) |
|---|---|---|
| **FastAPI / Starlette** | `-k uvicorn.workers.UvicornWorker` (ASGI) | `main:app` |
| **Flask** | *(sin `-k`)* (WSGI) | `app:app` |
| **Django** | *(sin `-k`)* (WSGI) | `miproyecto.wsgi:application` |

> `main:app` significa: en el archivo `main.py`, la variable de la app se llama `app`. Ajusta al nombre real de tu archivo y tu variable.

---

## Paso 3 — Correr con PM2 + gunicorn (¡y ver los logs EN VIVO!)

Un proceso por sí solo se muere cuando cierras el Terminal. Para mantener tu app corriendo usamos **PM2**, un gestor de procesos — la misma herramienta que usa producción, y **ya viene instalada** en tu hosting.

Primero crea la carpeta donde vivirá el socket (una sola vez):

```bash
mkdir -p ~/.sockets
```

Y arranca tu app con pm2 + gunicorn. Elige la línea según tu framework:

**FastAPI (ASGI):**
```bash
cd ~/miapi
pm2 start .venv/bin/gunicorn --name miapi --interpreter none -- \
  -k uvicorn.workers.UvicornWorker --workers 2 -b unix:$HOME/.sockets/miapi.sock main:app
pm2 save
```

**Flask (WSGI):**
```bash
cd ~/miapi
pm2 start .venv/bin/gunicorn --name miapi --interpreter none -- \
  --workers 2 -b unix:$HOME/.sockets/miapi.sock app:app
pm2 save
```

**Django (WSGI):**
```bash
cd ~/miproyecto
pm2 start .venv/bin/gunicorn --name miapi --interpreter none -- \
  --workers 2 -b unix:$HOME/.sockets/miapi.sock miproyecto.wsgi:application
pm2 save
```

> **¿Qué hace cada parte?**
> - `.venv/bin/gunicorn` → el gunicorn de TU entorno virtual.
> - `--name miapi` → el nombre con que verás tu app en pm2.
> - `--interpreter none` → le dice a pm2 que ejecute gunicorn directo (no con node).
> - `-b unix:$HOME/.sockets/miapi.sock` → publica la app en TU socket (usa el mismo nombre que `--name`).
> - lo último (`main:app`, `app:app`, `...wsgi:application`) → tu **entry point**.

### Mírala corriendo

```bash
pm2 list                               # tabla: id, nombre, estado, pid, uptime, cpu, mem
```

```
│ id │ name  │ mode │ pid    │ uptime │ ↺ │ status │ cpu │ mem    │
│ 0  │ miapi │ fork │ 716053 │ 3s     │ 0 │ online │ 0%  │ 25.0mb │
```

Comprueba que responde por su socket (aún sin dominio):

```bash
curl --unix-socket ~/.sockets/miapi.sock http://localhost/
```

### Los logs en vivo — lo mejor de todo

```bash
pm2 logs miapi
```

La consola se queda "pegada" mostrando cada línea nueva conforme sale (peticiones, prints, errores). Sales con **Ctrl-C** — y ojo: eso **solo deja de mirar los logs, NO detiene tu app.**

Más formas de ver logs:

```bash
pm2 logs                          # logs EN VIVO de TODAS tus apps a la vez
pm2 logs miapi --lines 200        # 200 líneas de historial y sigue en vivo
pm2 logs miapi --err              # SOLO errores (stderr), en vivo
pm2 logs miapi --nostream --lines 50   # imprime las últimas 50 y sale
pm2 flush miapi                   # vacía los archivos de log de esa app
```

**Dashboard interactivo** (CPU, memoria y logs en una pantalla, se sale con `q`):

```bash
pm2 monit
```

Archivos físicos de log:

```
~/.pm2/logs/miapi-out.log     (stdout — lo normal)
~/.pm2/logs/miapi-error.log   (stderr — errores)
```

### Controlar la app

```bash
pm2 restart miapi                # reinicio duro (úsalo tras cada cambio de código)
pm2 reload miapi                 # recarga suave
pm2 stop miapi                   # detiene pero lo deja en la lista
pm2 delete miapi                 # lo quita de PM2 por completo
```

> **Tras cambiar tu código:** `git pull` (o sube tus cambios) → si cambiaron dependencias `./.venv/bin/pip install -r requirements.txt` → `pm2 restart miapi`. Python no necesita "compilar".

---

## Paso 4 — Que sobreviva a los reinicios del servidor

Si el servidor se reinicia, quieres que tu app vuelva sola. Son **dos piezas**:

**A) Un servicio de arranque (lo activa una sola vez el administrador).**
Como cliente no eres root, así que PM2 no puede instalar el servicio de arranque por ti — pero **te imprime la línea exacta** que el admin debe correr:

```bash
pm2 startup
```

Verás algo así:

```
[PM2] To setup the Startup Script, copy/paste the following command:
sudo env PATH=$PATH:... pm2 startup systemd -u TU_USUARIO --hp /home/TU_USUARIO
```

**Copia esa línea `sudo ...` y pásasela a soporte de MBHostCloud.** Se corre una sola vez y queda listo para siempre.

**B) Guardar tu lista de apps (esto lo haces tú, sin root).**

```bash
pm2 save
```

Esto congela la lista actual (`Successfully saved in ~/.pm2/dump.pm2`). En el próximo arranque del servidor, PM2 resucita exactamente esas apps. Repite `pm2 save` cada vez que agregues, quites o cambies tus apps.

> **Cuidado:** para persistir usa siempre `pm2 save`. No intentes reiniciar el servicio con `systemctl` mientras tu daemon de PM2 está vivo. Si alguna vez necesitas forzarlo, primero `pm2 kill` y luego que el admin lo arranque.

---

## Paso 5 — Enlazar tu dominio/subdominio a tu app

Falta lo último: que cuando alguien entre a `api.tudominio.com`, el servidor reenvíe (reverse-proxy) el tráfico a tu app. **Esto NO lo configuras a mano** — en MBHostCloud el enlace se hace desde el **panel**, y nosotros apuntamos tu dominio a tu **socket** de forma segura.

> ⚠️ En el hosting compartido, "Custom HTTPD Configurations" y el `.htaccess` con proxy (`[P]`) están **deshabilitados a propósito, por seguridad** — así ningún cliente puede tocar (ni espiar) la configuración de otro. Por eso el enlace se hace por el panel; **no** editando configs de Apache a mano.

### Cómo enlazar

1. Crea tu **dominio o subdominio** en DirectAdmin (**Account Manager → Domain Setup**), si aún no existe.
2. Arranca tu app en pm2 (Paso 3), escuchando en su socket.
3. En tu **panel de MBHostCloud**, abre la sección **"Python App"**, elige tu **app** (de las que tengas en pm2) y tu **dominio/subdominio** → botón **Enlazar**.
   > *El **entry point** (`main:app`, `app:app`, `…wsgi:application`) y el tipo (ASGI/WSGI) se detectan solos de tu comando gunicorn — no los escribes.* El enlace es **casi instantáneo**.

**Alternativa por Terminal (CLI):** también puedes enlazar desde tu consola:

```bash
mbpython api.tudominio.com ~/miapi main:app miapi
mbpython --status          # ver el resultado del enlace
```
(Django: `mbpython api.tudominio.com ~/miproyecto miproyecto.wsgi:application miapi`.)

En segundos tu dominio queda sirviendo tu app **por socket** (sin puerto abierto), con **SSL**. Nosotros configuramos el reverse-proxy; **tú solo mantienes tu app viva con pm2 y ves tus logs.** 🎉

> *(Mientras habilitamos la sección "Python App" en tu panel, escríbenos tu dominio + carpeta de la app a **support@mbhostcloud.com** y lo activamos en el momento.)*

---

## Ejemplo completo (FastAPI, listo para copiar)

Un mini proyecto para probar todo el flujo de punta a punta:

```python
# ~/miapi/main.py
from fastapi import FastAPI

app = FastAPI(title="Mi API")

@app.get("/")
def root():
    return {"ok": True, "msg": "Hola desde mi API Python en MBHostCloud"}

@app.get("/health")
def health():
    return {"status": "up"}
```

```
# ~/miapi/requirements.txt
fastapi
gunicorn
uvicorn
```

Arrancarlo y verlo:

```bash
cd ~/miapi
python3 -m venv .venv
./.venv/bin/pip install -r requirements.txt
mkdir -p ~/.sockets
pm2 start .venv/bin/gunicorn --name miapi --interpreter none -- \
  -k uvicorn.workers.UvicornWorker --workers 2 -b unix:$HOME/.sockets/miapi.sock main:app
pm2 save
pm2 logs miapi
```

Enlaza `api.tudominio.com` desde el panel (**"Python App"**) o con `mbpython api.tudominio.com ~/miapi main:app miapi`, abre `https://api.tudominio.com` en tu navegador y verás el JSON de tu app; en `pm2 logs miapi` verás la petición aparecer en vivo. 🎉

---

## Tips y solución de problemas

**Tras cada cambio de código:**
```bash
cd ~/miapi
git pull
./.venv/bin/pip install -r requirements.txt   # solo si cambiaron dependencias
pm2 restart miapi
pm2 logs miapi                                  # confirma que arrancó sin errores
```

**Ver rápido si tu app está viva:**
```bash
pm2 list             # ¿status "online"? bien. ¿"errored" o "stopped"? revisa logs.
pm2 logs miapi --err --lines 50 --nostream
```

**`errored` apenas arranca:** casi siempre el **entry point** está mal (el archivo o el nombre de la variable). Revisa que `modulo:objeto` apunte a tu app real y míralo:
```bash
pm2 logs miapi --err
```

**Error 502 (Bad Gateway):** Apache no encuentra tu app. Suele ser:
- Tu app no está corriendo → `pm2 list` (¿"online"?).
- El socket del enlace no coincide con el de tu gunicorn → deben ser el mismo archivo (`~/.sockets/<name>.sock`).

**`ModuleNotFoundError` al arrancar:** falta una dependencia en tu `.venv`. Instálala:
```bash
./.venv/bin/pip install <paquete>
pm2 restart miapi
```

**Django — archivos estáticos (`/static/`):** corre `python manage.py collectstatic` y sirve `STATIC_ROOT`; para el admin/estáticos avísanos al enlazar y lo ajustamos.

**El SSL no renueva:** el sistema ya deja habilitado `/.well-known/` en tu proxy; si tienes dudas, escríbenos.

---

## En resumen (copy-paste)

```bash
# 1) Subir tu app (fuera de public_html)
cd ~ && git clone TU_REPO miapi && cd miapi

# 2) Entorno virtual + dependencias (Python 3 y pm2 YA vienen listos)
python3 -m venv .venv
./.venv/bin/pip install -r requirements.txt gunicorn uvicorn   # uvicorn solo si es FastAPI

# 3) Correr con pm2 + gunicorn y ver logs en vivo
mkdir -p ~/.sockets
pm2 start .venv/bin/gunicorn --name miapi --interpreter none -- \
  -k uvicorn.workers.UvicornWorker --workers 2 -b unix:$HOME/.sockets/miapi.sock main:app
#   Flask:  quita el "-k ..." y usa  app:app
#   Django: quita el "-k ..." y usa  miproyecto.wsgi:application
pm2 logs miapi           # <-- LOGS EN VIVO (Ctrl-C para salir, la app sigue)

# 4) Que sobreviva reinicios
pm2 save

# 5) Enlazar el dominio: por el panel (sección "Python App"),
#    o por Terminal:   mbpython api.tudominio.com ~/miapi main:app miapi
#    o pídelo a support@mbhostcloud.com
```

¡Listo! Tu app Python corre bajo tu control, sobrevive reinicios, y puedes ver todo lo que hace en tiempo real — **sin haber tocado tu código.** Bienvenido al control total. 🚀

---

## ¿Necesitas ayuda?

Esta guía es **exclusiva para clientes de MBHostCloud®**. Para el único paso que requiere administrador (el `pm2 startup` que activa el arranque en boot) o para cualquier duda del despliegue, escríbenos a **support@mbhostcloud.com** y te ayudamos.

<sub>MBHostCloud® — un servicio de MarBust Technology Company. Hosting con control real y logs en vivo.</sub>
