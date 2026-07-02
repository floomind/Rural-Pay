# Rural Pay — Arquitectura Técnica

## 1. Visión General

Plataforma de pagos multi-tenant para gestionar la cobranza de obligaciones de dos clientes principales bajo políticas independientes. Los deudores reciben un QR con un link único firmado que los lleva a una landing pública donde eligen plan de cuotas / beneficio y abonan mediante checkout embebido (sin abandonar el sitio). Los fondos se acreditan al CBU de cada cliente. Rural Pay cobra un fee de gestión configurable por cliente.

## 2. Stack Técnico

| Capa | Tecnología | Razón |
|---|---|---|
| Backend | **FastAPI 0.115+** | Async nativo, OpenAPI auto, validación Pydantic |
| ORM | **SQLModel 0.0.22+** | Pydantic + SQLAlchemy, type safety end-to-end |
| DB | **PostgreSQL 16** | ACID, robustez transaccional crítica para fintech |
| Migraciones | **Alembic** | Versionado de schema |
| Auth | **JWT (RS256) + OAuth2 Password Flow** | Estándar, asimétrico, refresh tokens |
| Hashing | **argon2-cffi** | OWASP recomendado, resistente a GPU |
| Cache / Queue | **Redis 7** | Sessions, rate limit, jobs async |
| Frontend | **Jinja2 + HTMX 2 + Alpine.js 3 + Tailwind CSS 3** | Interactividad sin SPA, productividad alta |
| Charts | **Chart.js 4** | Liviano, profesional |
| Pasarela | **Mercado Pago Checkout Bricks** (adapter base para multi-gateway) | Mejor SDK Python + embebido sin redirección |
| QR | **segno** | QR puro Python, sin deps nativas |
| Server | **Uvicorn + Gunicorn** | Producción |
| Tests | **pytest + pytest-asyncio + httpx** | Estándar |
| Lint/Format | **ruff + black + mypy** | Calidad de código |
| Contenedor | **Docker + docker-compose** | Reproducibilidad |

## 3. Arquitectura por capas

```
┌────────────────────────────────────────────────────────────────┐
│  Web Layer (Jinja2 + HTMX + Tailwind)                          │
│  - /                       landing                             │
│  - /login                  auth                                │
│  - /panel/*                dashboards (gestor / cliente)        │
│  - /pagar/{token}          landing pública del deudor          │
├────────────────────────────────────────────────────────────────┤
│  API Layer (FastAPI Routers, /api/v1)                          │
│  - auth, users, clients, debtors, obligations,                 │
│    payments, webhooks, metrics                                 │
├────────────────────────────────────────────────────────────────┤
│  Service Layer (lógica de negocio)                             │
│  - AuthService, ObligationService, PaymentService,             │
│    MetricsService, QrService, AuditService                     │
├────────────────────────────────────────────────────────────────┤
│  Integration Layer (adapter pattern)                           │
│  - PaymentGatewayBase                                          │
│  - MercadoPagoGateway, ModoGateway (stub)                      │
├────────────────────────────────────────────────────────────────┤
│  Data Layer (SQLModel + AsyncSession)                          │
│  - PostgreSQL                                                  │
└────────────────────────────────────────────────────────────────┘
```

## 4. Modelo de datos (resumen)

- **User**: gestores Rural Pay y referentes de cada cliente. Campos: id, email, password_hash, full_name, role, client_id (nullable), is_active, created_at.
- **Role** (enum): `RURAL_PAY_ADMIN`, `RURAL_PAY_OPERATOR`, `CLIENT_ADMIN`, `CLIENT_VIEWER`.
- **Client**: los dos clientes destino. Campos: id, legal_name, tax_id, cbu, alias_cbu, fee_percentage, payment_policy (JSONB), branding (JSONB), is_active.
- **Debtor**: deudor de un cliente. Campos: id, client_id, tax_id, full_name, email, phone.
- **Obligation**: deuda concreta. Campos: id, client_id, debtor_id, external_reference, principal_amount, currency, due_date, status (`pending`, `in_progress`, `paid`, `expired`, `cancelled`), qr_token (firmado), metadata (JSONB).
- **PaymentPlan**: plan de cuotas o beneficio aplicable a una obligación (lo elige el deudor). Campos: id, obligation_id, type (`single`, `installments`, `discount`), installments_count, surcharge_pct, discount_pct, total_amount.
- **Payment**: intento o pago concreto. Campos: id, obligation_id, plan_id, gateway, gateway_payment_id, amount, status, paid_at, payer_data (JSONB), raw_response (JSONB).
- **Transaction**: ledger interno. Campos: id, payment_id, type (`debit`, `credit`, `fee`), amount, account (cliente|rural_pay), created_at.
- **AuditLog**: append-only. Campos: id, actor_id, action, entity_type, entity_id, payload (JSONB), ip, user_agent, created_at.

## 5. Multi-tenancy

- Aislamiento por `client_id` a nivel de fila.
- Middleware/dependency `get_current_tenant()` que infiere el cliente desde el usuario logueado.
- Queries siempre filtradas por `client_id` cuando el usuario no es `RURAL_PAY_ADMIN`.
- Política RLS (Row Level Security) a futuro en PostgreSQL.

## 6. Flujo de cobro QR

1. Gestor Rural Pay (o sistema del cliente vía API) crea una `Obligation` para un `Debtor`.
2. Se genera un `qr_token` firmado (HS256, exp 30 días) que apunta a `/pagar/{token}`.
3. Sistema renderiza QR (segno) descargable / imprimible / enviable por email.
4. Deudor escanea el QR → landing pública `/pagar/{token}` (sin login).
5. Landing muestra: monto original, vencimiento, planes disponibles (según política del cliente).
6. Deudor elige plan → backend crea `PaymentPlan` y `Preference` en Mercado Pago.
7. Se embebe **Payment Brick** en la misma página → el deudor paga sin redireccionar.
8. Webhook IPN de Mercado Pago llega a `/api/v1/webhooks/mercadopago` → verifica firma → actualiza `Payment` y `Obligation`.
9. Transacciones internas se asientan en `Transaction` (crédito al cliente, débito por fee a Rural Pay).
10. Dashboard refleja métricas en tiempo real (polling cada 15s vía HTMX o WebSocket más adelante).

## 7. Seguridad

- JWT RS256 con par de llaves; refresh tokens almacenados en DB con rotación.
- Argon2id para passwords (parámetros: time_cost=3, memory_cost=64MB, parallelism=4).
- Cookies HttpOnly + SameSite=Lax + Secure en producción.
- CSRF tokens en formularios HTMX.
- Rate limiting (slowapi) en login, webhooks y endpoints públicos.
- Verificación de firma del webhook de Mercado Pago (`x-signature` HMAC).
- Idempotency keys en creación de pagos.
- Audit log append-only de toda acción sensible.
- Validación estricta con Pydantic en cada borde.
- HTTPS obligatorio en producción, HSTS.
- Secrets en `.env` (gitignored) y, en producción, en vault.

## 8. Dashboards

- **Rural Pay Admin**: visión consolidada por ambos clientes. KPIs: recaudado hoy/semana/mes/año, fees acumulados, top deudores morosos, conversión QR → pago, tiempo promedio de cobro.
- **Cliente**: vista limitada a su `client_id`. Mismos KPIs pero scope acotado.
- Cálculos vía SQL agregado (`date_trunc`, `sum`, `count`) + cache Redis 60s para queries pesadas.
- Endpoints `/api/v1/metrics/*` devuelven JSON, el frontend lo grafica con Chart.js.

## 9. Estructura de carpetas

```
RURAL PAY/
├── app/
│   ├── main.py                  # FastAPI app + lifespan
│   ├── config.py                # Pydantic Settings (.env)
│   ├── database.py              # Engine + sessions async
│   ├── security.py              # JWT, hashing, deps
│   ├── dependencies.py          # FastAPI deps comunes
│   ├── models/                  # SQLModel tables
│   ├── schemas/                 # Pydantic IO
│   ├── api/v1/                  # Routers REST
│   ├── services/                # Lógica de negocio
│   ├── integrations/            # Gateways de pago (adapter)
│   ├── web/                     # Rutas HTML + templates + static
│   └── utils/                   # Helpers
├── alembic/                     # Migraciones
├── tests/                       # pytest
├── scripts/                     # PowerShell (setup/run/test/migrate)
├── docs/                        # Documentación
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
├── .env.example
└── README.md
```

## 10. Roadmap del MVP (lo entregado en este pase)

- [x] Scaffolding completo y reproducible (Docker + Postgres + Redis).
- [x] Autenticación JWT + roles + middleware multi-tenant.
- [x] Modelos completos + migración inicial.
- [x] Adapter pattern para gateways (Mercado Pago activo, MODO stub).
- [x] Endpoints REST para clientes, deudores, obligaciones, pagos, webhooks, métricas.
- [x] Landing pública del deudor con Payment Brick embebido.
- [x] Generación de QR firmado.
- [x] Dashboard para Rural Pay y dashboard para cliente.
- [x] Scripts PowerShell para Windows.
- [x] Tests básicos.

## 11. Roadmap futuro (post-MVP)

- 2FA TOTP para gestores.
- Conciliación bancaria automática.
- Notificaciones por email/WhatsApp al deudor (recordatorios).
- Plan de cuotas con financiación propia del cliente.
- Reportes PDF descargables.
- WebSocket para actualización push del dashboard.
- Row Level Security a nivel de PostgreSQL.
- Activar gateway MODO en producción si comisión efectiva resulta menor.
