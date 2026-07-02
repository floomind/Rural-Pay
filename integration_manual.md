# Manual de Integración — Rural Pay

Este documento es la guía completa para que un cliente (entidad sindical,
obra social, gremio, asociación) integre su ERP/plataforma con Rural Pay.

Rural Pay es una capa de **trazabilidad de cobranza**: vos seguís cobrando
en tus CBUs, pero ahora todo el ciclo queda auditado, automatizado y con
notificaciones en tiempo real.

---

## 1. Modelo de integración

```
        ┌─────────────────────────────┐
        │   Tu sistema (SAP / Web)    │
        └──────────────┬──────────────┘
                       │
            INBOUND    │   OUTBOUND
        (push acts,    │  (webhooks de
         debtors)      │   pagos)
                       │
        ┌──────────────▼──────────────┐
        │        Rural Pay            │
        └─────────────────────────────┘
```

Hay **dos canales de comunicación**, ambos REST + JSON + HMAC:

| Canal | Dirección | Quién inicia |
|---|---|---|
| Inbound | Tu ERP → Rural Pay | Vos hacés `POST` cuando hay un cambio |
| Outbound | Rural Pay → Tu ERP | RP hace `POST` cuando un pago se concilia |

---

## 2. Endpoints inbound (Tu ERP llama a Rural Pay)

Base URL: `https://api.ruralpay.com.ar`

Todos los endpoints requieren los headers:

```
Content-Type: application/json
X-RP-Signature: <hex(HMAC-SHA256(body, tu_inbound_secret))>
X-Idempotency-Key: <opcional — UUID por operación para evitar duplicados>
```

### 2.1 Alta/actualización de empresa deudora

```
POST /api/v1/integrations/{short_name}/debtors/upsert
```

Donde `{short_name}` es tu identificador (ej. `UATRE`, `OSPRERA`).

**Body**:
```json
{
  "cuit": "30-12345678-9",
  "razon_social": "AGRO GANADERA EFETE S.R.L.",
  "email": "contacto@agroganadera.com",
  "telefono": "+54 11 4444-5555",
  "direccion": "Av. del Campo 1234, Pcia. Buenos Aires",
  "external_id": "ERP-DEBTOR-87123",
  "metadata": {"any": "extra fields"}
}
```

**Respuesta**:
```json
{
  "ok": true,
  "event_id": 12345,
  "rural_pay_id": 678,
  "message": "Deudor created"
}
```

### 2.2 Alta/actualización de acta (obligación)

```
POST /api/v1/integrations/{short_name}/acts/upsert
```

**Body**:
```json
{
  "external_reference": "ACTA-2026-00781",
  "debtor_cuit": "30-12345678-9",
  "items": [
    {"concept_code": "cuota_sindical", "principal_amount": "1524785.24"},
    {"concept_code": "cuota_solidaria", "principal_amount": "265785.12"},
    {"concept_code": "cuota_sepelio", "principal_amount": "2854581.78"}
  ],
  "due_date": "2026-09-30",
  "description": "Acta de regularización Q3-2026",
  "period_from": "2026-08",
  "period_to": "2026-08",
  "metadata": {"agente": "AG-09812"}
}
```

**Notas**:
- El deudor debe existir previamente (sino mandalo en debtors/upsert primero).
- Si re-mandás el mismo `external_reference`, se actualiza (los items se reemplazan).
- Los `concept_code` deben coincidir con los que sincronizaste vía `concepts/sync`.

### 2.3 Sincronización de conceptos cobrables

```
POST /api/v1/integrations/{short_name}/concepts/sync
```

**Body**:
```json
{
  "code": "cuota_sindical",
  "name": "Cuota Sindical UATRE",
  "cbu": "0110599520000026026486",
  "alias_cbu": "UATRE.SINDICAL",
  "bank_name": "Banco Nación",
  "is_mandatory": true,
  "is_active": true
}
```

Cada concepto se sincroniza con un POST separado.

### 2.4 Webhook genérico

```
POST /api/v1/integrations/{short_name}/webhook
```

Para eventos que no encajan en los anteriores:

```json
{
  "event_type": "debtor.address_changed",
  "payload": {"cuit": "30-12345678-9", "new_address": "..."}
}
```

---

## 3. Notificaciones outbound (Rural Pay llama a tu ERP)

Cuando un pago se concilia (validado el comprobante por OCR e IA), Rural Pay
te notifica vía POST al endpoint que vos configures.

Vas a recibir requests con headers:

```
Content-Type: application/json
X-RP-Signature: <hex(HMAC-SHA256(body, tu_outbound_secret))>
```

### 3.1 Pago conciliado

`POST {tu_base_url}/webhooks/rural-pay/payment-reconciled`

**Body**:
```json
{
  "event_type": "payment.reconciled",
  "rural_pay_payment_id": 9871,
  "rural_pay_obligation_id": 4421,
  "external_reference": "ACTA-2026-00781",
  "debtor_cuit": "30-12345678-9",
  "concept_code": "cuota_sindical",
  "amount": "1524785.24",
  "fee_amount": "160102.45",
  "net_to_client": "1364682.79",
  "payer_full_name": "AGRO GANADERA EFETE S.R.L.",
  "payer_cuit": "30-12345678-9",
  "payer_bank": "Galicia",
  "bank_statement_ref": "459872613",
  "reconciled_at": "2026-09-15T14:32:18.124Z",
  "comprobante_url": "https://api.ruralpay.com.ar/files/signed/abc123.pdf"
}
```

**Tu endpoint debe responder con 200**:
```json
{
  "received": true,
  "client_reference": "ERP-PAGO-887211"
}
```

### 3.2 Pago disputado

`POST {tu_base_url}/webhooks/rural-pay/payment-disputed`

Cuando un pago fue rechazado por la IA (comprobante adulterado, monto incoherente).

---

## 4. Política de reintentos

Si tu endpoint responde **5xx**, **timeout** o **connection error**, Rural Pay reintenta con backoff:

| Intento | Espera |
|---|---|
| 1 | inmediato |
| 2 | 1 minuto |
| 3 | 5 minutos |
| 4 | 15 minutos |
| 5 | 1 hora |
| 6 | 4 horas |

Tras 6 intentos fallidos → **dead letter**. Tu administrador en RP puede ver
el evento en el panel y reintentarlo manualmente.

Si tu endpoint responde **4xx** (incluido 400, 401, 403, 422), se considera
error **permanente** y el evento va directo a dead letter (no reintenta).

---

## 5. Firma HMAC

Cada request se firma con HMAC-SHA256 para garantizar autenticidad.

**Cómo firmar (cliente envía a RP)**:

```python
import hmac, hashlib
secret = "tu_inbound_secret_de_RP"
body = json.dumps(payload).encode("utf-8")
signature = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
# Enviar header: X-RP-Signature: <signature>
```

**Cómo verificar (cliente recibe de RP)**:

```python
expected = hmac.new(outbound_secret.encode(), body, hashlib.sha256).hexdigest()
if not hmac.compare_digest(expected, request.headers["X-RP-Signature"]):
    return 401
```

---

## 6. Onboarding técnico

Para empezar:

1. RP genera tu `client_short_name` (ej. `UATRE`).
2. RP genera y te entrega tu `inbound_secret` (HMAC para validar tus POSTs).
3. Vos nos compartís:
   - Tu `base_url` para outbound (ej. `https://erp.uatre.org.ar`).
   - Tu `outbound_secret` (HMAC para que validés nuestros POSTs).
   - Tipo de auth que requerís: API Key, OAuth2 Client Credentials, o Basic.
4. Cargás tus conceptos cobrables vía `concepts/sync`.
5. Empezás a sincronizar deudores y actas.

Mientras estés en homologación, RP queda en modo `mock`: te logueamos lo que
te mandaríamos pero no llamamos a tu ERP. Cuando estés listo, RP cambia a
`webrest` (o `sap_rest`, según corresponda) y la integración queda activa.

---

## 7. Ejemplos curl

### Subir un acta

```bash
BODY='{"external_reference":"ACTA-2026-00781","debtor_cuit":"30-12345678-9","items":[{"concept_code":"cuota_sindical","principal_amount":"1524785.24"}]}'
SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "tu_secret" -hex | sed 's/^.* //')

curl -X POST https://api.ruralpay.com.ar/api/v1/integrations/UATRE/acts/upsert \
  -H "Content-Type: application/json" \
  -H "X-RP-Signature: $SIG" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d "$BODY"
```

### Probar tu endpoint outbound

```bash
# Desde el panel admin de RP: POST /api/v1/integrations-admin/{client_id}/test
# Te hace un health_check al base_url configurado.
```

---

## 8. Soporte

Cualquier duda sobre la integración: **integrations@ruralpay.com.ar**.

Para urgencias durante onboarding podés contactar a tu agente asignado.
