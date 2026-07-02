<div align="center">

# 💳 Rural Pay

### Trazabilidad y conciliación de cobranzas de aportes rurales (multi-tenant)

_Plataforma web que identifica, archiva y concilia las cobranzas de obligaciones del sector rural para clientes con políticas independientes. **Rural Pay no procesa pagos**: la plata sigue yendo del deudor al CBU del cliente. Lo que aporta es identificar inequívocamente cada transferencia con un **código de referencia único**, archivar los comprobantes y conciliar contra el extracto bancario (manual o import CSV). El fee de gestión se factura mensualmente a cada cliente._

<br/>

![Python](https://img.shields.io/badge/Python_3.12-3776AB?style=for-the-badge&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)
![SQLModel](https://img.shields.io/badge/SQLModel-7E56C2?style=for-the-badge&logo=python&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL_16-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis_7-DC382D?style=for-the-badge&logo=redis&logoColor=white)

![HTMX](https://img.shields.io/badge/HTMX-3366CC?style=for-the-badge&logo=htmx&logoColor=white)
![Alpine.js](https://img.shields.io/badge/Alpine.js-8BC0D0?style=for-the-badge&logo=alpinedotjs&logoColor=black)
![Tailwind CSS](https://img.shields.io/badge/Tailwind-06B6D4?style=for-the-badge&logo=tailwindcss&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)

</div>

---

## 📸 Capturas

<!-- TODO: subir capturas a docs/screenshots/ y reemplazar los enlaces. -->

<div align="center">

| Landing pública | Scoring de cumplimiento | Panel / Conciliación |
|:---:|:---:|:---:|
| _(captura aquí)_ | _(captura aquí)_ | _(captura aquí)_ |

</div>

---

## ✨ Características destacadas

- 🏢 **Multi-tenant** con políticas independientes por cliente y roles diferenciados.
- 🏦 **Cobranza CBU directo:** Rural Pay no toca los fondos; identifica cada transferencia con **código de referencia único** y concilia.
- 🧾 **Conciliación manual o por import CSV** del extracto bancario, con matching automático por código.
- 🏅 **Scoring de cumplimiento:** cada obligación pagada en término suma puntos y sube el nivel del deudor (Inicial → Bueno → Muy bueno → Excelente), desbloqueando mejores descuentos, cuotas más extendidas y condiciones especiales.
- 🤖 **Asistente / chatbot integrado:** widget de ayuda con FAQ, respuestas rápidas y derivación a un asesor por WhatsApp.
- 📒 **Ledger + facturación mensual** de fees por cliente (`/api/v1/billing/monthly`).
- 📊 **Dashboards en tiempo real** de recaudación, cupones y conciliaciones.

---

## 🧱 Stack

- **Python 3.12** + **FastAPI 0.115** + **SQLModel** + **PostgreSQL 16** + **Redis 7**
- **Almacenamiento local** de comprobantes con SHA-256 (swappable a S3 después)
- **JWT + Argon2id** para auth, **HTMX + Alpine.js + Tailwind CSS** para el frontend
- **Alembic** para migraciones · **pytest** para tests · **Docker** para entorno reproducible

Detalle técnico en `docs/arquitectura.md` y modelo operativo en `docs/modelo-trazabilidad.md`.

---

## 🛠️ Requisitos

- Python 3.12+
- PowerShell (Windows)
- Docker Desktop (para Postgres + Redis)

## 🚀 Setup rápido

```powershell
# 1) Levantar Postgres + Redis
docker compose up -d db redis

# 2) Crear venv e instalar dependencias
.\scripts\setup.ps1

# 3) Editar .env (al menos SECRET_KEY y QR_TOKEN_SECRET)
notepad .env

# 4) Aplicar migraciones
.\scripts\migrate.ps1

# 5) Datos de ejemplo (opcional pero recomendado)
.\scripts\seed.ps1

# 6) Arrancar la app
.\scripts\run.ps1
```

Abrir en el navegador:

- Landing: http://localhost:8000
- Login: http://localhost:8000/login
- Docs API: http://localhost:8000/api/docs

## 👥 Usuarios sembrados (demo local)

| Rol | Email | Password |
|---|---|---|
| Rural Pay Admin | `admin@ruralpay.local` | `RuralPay2026!` |
| Cliente 1 Admin | `admin@coopsur.local` | `CoopSur2026!` |
| Cliente 2 Admin | `admin@estanciasnorte.local` | `Estancias2026!` |

> Credenciales de ejemplo para entorno local. Cambiar antes de producción.

---

## 🗂️ Estructura

```
app/
  main.py             FastAPI + middlewares
  config.py           Pydantic Settings (.env)
  database.py         Engine async + sessions
  security.py         JWT, hashing, firma de QR
  dependencies.py     Auth + multi-tenant
  models/             SQLModel
  schemas/            Pydantic IO
  api/v1/             Routers REST
  services/           Lógica de negocio
  integrations/       Gateways (MP + MODO)
  web/                Vistas HTML, templates, estáticos
alembic/              Migraciones
scripts/              setup / run / migrate / test / seed
tests/                pytest
docs/                 Documentación
```

## ⌨️ Comandos útiles

```powershell
.\scripts\setup.ps1                  # Setup inicial
.\scripts\run.ps1                    # Arrancar dev server
.\scripts\migrate.ps1                # Aplicar migraciones (upgrade head)
.\scripts\migrate.ps1 -New "msg"     # Crear migración nueva (autogenerate)
.\scripts\seed.ps1                   # Cargar datos de ejemplo
.\scripts\test.ps1                   # Correr tests
```

---

## 🔄 Probar el flujo end-to-end

1. Loguearse como admin Rural Pay en `/login`.
2. Ir a `/panel/obligaciones`.
3. Copiar el **Link** de una obligación (URL `/pagar/{token}`).
4. Abrirla en otra pestaña en modo incógnito (simulamos al deudor):
   - Paso 1: elegir plan de pago (contado, cuotas).
   - Paso 2: ver CBU, alias y **código de referencia destacado** → copiar el código.
   - Paso 3: completar datos de pagador, subir comprobante (PDF/imagen) y declarar.
5. Volver al panel admin → `/panel/conciliacion` → la declaración aparece en la cola.
6. Opciones:
   - **Aprobar manualmente** (revisar comprobante adjunto, click "✓ Aprobar y conciliar").
   - **Importar CSV** del extracto bancario para matching automático por código.
7. La obligación pasa a `paid`, la `Transaction` queda asentada, y el dashboard refleja el ingreso.
8. A fin de mes: `/api/v1/billing/monthly?year=2026&month=5` devuelve la liquidación que Rural Pay debe facturar a cada cliente.

---

## 🔐 Seguridad

- Argon2id para passwords (OWASP recomendado).
- JWT con expiración corta + refresh tokens en cookies HttpOnly.
- Tokens de QR firmados con `itsdangerous` (no es posible adivinar el ID de una obligación).
- Comprobantes adjuntos archivados con hash SHA-256 para auditoría.
- Validación de MIME type y tamaño máximo en upload de comprobantes.
- Rate limit en endpoints sensibles (slowapi).
- Audit log append-only de toda acción crítica.
- CORS, HTTPS, HSTS recomendados en producción.

---

## 🚦 Roadmap

- Notificaciones email al deudor (confirmación de conciliación, recordatorios).
- OCR sobre comprobantes para extraer monto/fecha/CBU automáticamente.
- Open Banking para conciliación en tiempo real (Bind, Alkemy, Itaú Conecta).
- Reportes PDF descargables.
- 2FA TOTP para operadores.
- Activar adapter de pasarela (MODO / Mercado Pago) para ofrecer pago por QR en la misma plataforma.

---

## 👤 Autor

**Agustín Formica** — Diseño y desarrollo full-stack (arquitectura, backend, modelo de negocio y frontend).

[![Email](https://img.shields.io/badge/Email-agustin.formica@gmail.com-EA4335?style=flat-square&logo=gmail&logoColor=white)](mailto:agustin.formica@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-agustín--formica-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/agustín-formica)

## 📄 Licencia

Propietario — Rural Pay (Agustín Formica)

