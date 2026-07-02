# Instalación en Windows — guía paso a paso

Esta guía cubre desde cero hasta tener Rural Pay corriendo en `http://localhost:8000`.

## Paso 1 — Diagnóstico inicial

Abrí PowerShell en la carpeta del proyecto y corré:

```powershell
.\scripts\check.ps1
```

Esto te dice exactamente qué falta instalar. Si te dice `Resolve los items en rojo`, seguí con los pasos de abajo.

---

## Paso 2 — Instalar Python 3.12

### Opción recomendada: instalador oficial

1. Descargar desde **https://www.python.org/downloads/windows/** la versión **Python 3.12.x** (Windows installer 64-bit).
2. Ejecutar el instalador. **MUY IMPORTANTE**: tildar **`Add python.exe to PATH`** en la primera pantalla.
3. Elegir "Install Now".
4. Cerrar y reabrir PowerShell.
5. Verificar: `python --version` → debe decir `Python 3.12.x`.

### Si te aparece "El ejecutable especificado no es una aplicación válida"

Es el "alias fantasma" de Microsoft Store. Resolución:

1. Abrir **Configuración de Windows** → **Aplicaciones** → **Configuración avanzada de aplicaciones** → **Alias de ejecución de aplicaciones**.
2. Desactivar las dos entradas que dicen `python.exe` y `python3.exe`.
3. Cerrar y reabrir PowerShell.
4. Volver a verificar con `python --version`.

---

## Paso 3 — Instalar Docker Desktop

Docker es necesario para correr Postgres y Redis en local sin tener que instalarlos a mano.

1. Descargar **Docker Desktop for Windows** desde **https://www.docker.com/products/docker-desktop/**.
2. Ejecutar el instalador. Aceptar habilitar WSL 2 (lo va a hacer solo).
3. **Reiniciar la PC** cuando lo pida.
4. Abrir **Docker Desktop** (queda en la barra de tareas como un ícono de ballena).
5. Esperar a que diga "Docker Desktop is running".
6. Verificar en PowerShell: `docker --version` → debe responder con la versión.

---

## Paso 4 — Habilitar ejecución de scripts PowerShell

Si tu PowerShell tiene la política restringida, abrí PowerShell **como administrador** una sola vez y corré:

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

Confirmá con `S` cuando pregunte.

---

## Paso 5 — Arrancar Rural Pay

Cerrá y reabrí PowerShell normal (no admin). Posicioná en la carpeta del proyecto:

```powershell
cd "D:\RURAL PAY\RURAL PAY"
```

Y ahora sí, los 6 pasos secuenciales:

```powershell
.\scripts\check.ps1                    # diagnostico final - debe estar todo OK
docker compose up -d db redis          # 1. arrancar Postgres y Redis
.\scripts\setup.ps1                    # 2. crear venv e instalar deps Python
notepad .env                           # 3. cambiar SECRET_KEY y QR_TOKEN_SECRET (>=64 chars cada uno)
.\scripts\migrate.ps1                  # 4. aplicar migraciones de la DB
.\scripts\seed.ps1                     # 5. cargar datos de prueba
.\scripts\run.ps1                      # 6. levantar la app
```

Abrir en el navegador: **http://localhost:8000** y loguearse con `admin@ruralpay.local / RuralPay2026!`.

---

## Problemas frecuentes

### `pip install` falla con error de compilación de `psycopg2`

Necesitás las "Microsoft C++ Build Tools". Descargá desde **https://visualstudio.microsoft.com/visual-cpp-build-tools/** y al instalar elegí el workload "Desktop development with C++".

Como alternativa, el `pyproject.toml` ya usa `psycopg2-binary` (que viene pre-compilado), así que esto no debería pasar. Si pasa, abrí un issue.

### Docker compose no puede levantar Postgres (puerto 5432 ocupado)

Tenés otro Postgres corriendo localmente. Opciones:

- Pararlo: en PowerShell admin → `Stop-Service postgresql-x64-XX`.
- O cambiar el puerto en `docker-compose.yml` (línea `ports: 5432:5432` → `5433:5432`) y actualizar `DATABASE_URL` en `.env`.

### El navegador dice "Esta conexión no es privada" en localhost

Es normal. En desarrollo no usamos HTTPS. En producción sí (con nginx + Let's Encrypt).

### El webhook de Mercado Pago no llega a mi localhost

Mercado Pago necesita una URL pública. Para probar en local usá **ngrok**:

```powershell
# Instalar ngrok desde https://ngrok.com/download
ngrok http 8000
```

Te da una URL `https://xxxx.ngrok-free.app`. Esa URL la ponés en:

- `.env` → `APP_BASE_URL=https://xxxx.ngrok-free.app`
- Panel de developers de Mercado Pago → Notificaciones → URL del webhook: `https://xxxx.ngrok-free.app/api/v1/webhooks/mercadopago`

Reiniciá la app después de cambiar `.env`.
