# Modelo operativo: trazabilidad de cobranzas

**Rural Pay no es una pasarela de pagos.** Es un **sistema de trazabilidad y conciliación**. La plata no pasa por Rural Pay en ningún momento: el deudor transfiere directamente al CBU del cliente, igual que hoy. Rural Pay agrega lo que falta: **identificar inequívocamente a qué deuda corresponde cada transferencia**.

## Problema que resuelve

Hoy los clientes reciben transferencias con problemas:
- Sin CUIT en el concepto → imposible saber a qué deudor pertenecen.
- Importes incorrectos → no se sabe qué obligación cancelar.
- Sin trazabilidad → no hay forma de auditar el ciclo deuda → cobro → cierre.

## Solución en 4 piezas

### 1. Cupón digital con código único de referencia

Cada `Obligation` se crea con un `transfer_code` corto y único:

```
RP-A8K2J9-0047
```

Este código va impreso en el QR que recibe el deudor. La landing pública (`/pagar/{token}`) le muestra los datos exactos para transferir:

- CBU del cliente
- Alias CBU del cliente
- Importe exacto
- **Código de referencia destacado** que el deudor copia y pega en el concepto

### 2. Declaración del pago + comprobante obligatorio

El deudor toca **"Ya transferí"** y completa:

- CUIT, razón social, banco origen, CBU origen
- Importe declarado, fecha y hora
- **Comprobante adjunto obligatorio** (PDF/JPG/PNG/WEBP, máx 10 MB)
- Notas opcionales

Rural Pay crea un `Payment` con estado `declared` y guarda el comprobante en `storage/receipts/YYYY/MM/{uuid}.{ext}`.

### 3. Conciliación con el extracto bancario

El operador Rural Pay tiene 3 caminos en `/panel/conciliacion`:

#### 3.a — Conciliación manual

Para cada declaración pendiente: ver el comprobante, validar contra el home banking del cliente, marcar como **Aprobada** o **En disputa** con motivo.

#### 3.b — Import CSV del extracto (más práctico)

El operador (o el cliente directamente) sube el extracto del home banking en CSV. Rural Pay:

1. Lee cada fila del extracto.
2. Busca el `transfer_code` dentro del campo "concepto" de la fila.
3. Compara el monto contra el `Payment.amount` esperado.
4. Si matchea: marca como `reconciled` automáticamente.
5. Si hay mismatch de monto: lo deja en lista de conflictos para revisión.
6. Si no hay match: queda en lista de no-matcheadas.

Devuelve un reporte: `{ matched, unmatched, conflicts }`.

#### 3.c — Open Banking (fase 2)

API en tiempo real con el banco del cliente (Bind, Alkemy, Itaú Conecta, etc.). Requiere autorización del cliente firmada en su banco (NO requiere estatutos). Se implementa cuando haya volumen que lo justifique.

### 4. Estados del pago

```
EMITTED ────► DECLARED ────► RECONCILED   (camino feliz)
                 │
                 ├─► UNDER_REVIEW (operador investigando)
                 │
                 └─► DISPUTED (importe no coincide, sin comprobante, etc.)
                       │
                       └─► RECONCILED (resuelto a favor) o CANCELLED (resuelto contra)
```

## Modelo de negocio: facturación mensual al cliente

Como Rural Pay no toca los fondos, **no puede retener su comisión del pago**. La cobra a fin de mes con factura electrónica:

```
GET /api/v1/billing/monthly?year=2026&month=5
```

Devuelve por cada cliente: total recaudado en el mes, comisión que Rural Pay debe facturar, cantidad de transacciones.

## Aspectos regulatorios

- **No es PSPCP**: Rural Pay no procesa pagos, no necesita registrarse ante el BCRA.
- **No requiere estatutos del cliente**: solo CBU, alias y CUIT.
- **Servicio facturable**: "gestión de cobranzas y conciliación".
- **Ley 25.326 (Protección de Datos)**: los datos del deudor están en la plataforma. Aplica.
- **Audit log append-only**: toda acción crítica queda registrada.

## Flujo end-to-end

```
1. Operador Rural Pay   ──> Crea Obligacion (genera transfer_code + QR firmado)
                                   │
2. QR impreso/email     ────────────┘
                        │
3. Deudor escanea QR ───┘
                        │
4. Landing publica      ──> Muestra CBU + alias + importe + transfer_code
                        │
5. Deudor transfiere    ──> Desde home banking, pega transfer_code en el concepto
                        │
6. Plata cae en CBU     ──> Cuenta corriente del cliente (Rural Pay no toca un peso)
                        │
7. Deudor declara       ──> Sube comprobante + datos. Payment = DECLARED
                        │
8a. Operador concilia   ──> manual desde /panel/conciliacion. Payment = RECONCILED
8b. CSV import          ──> Cliente sube extracto, matching automatico. Payment = RECONCILED
                        │
9. Fee Rural Pay        ──> Se acumula en Transaction (FEE_RURAL_PAY)
                        │
10. Fin de mes          ──> Reporte billing/monthly -> Rural Pay factura al cliente
                        │
11. Cliente paga fee    ──> Transferencia separada a Rural Pay (fuera de la plataforma)
```

## Lo bueno de este modelo

- **Trazabilidad 100%** del ciclo deuda → transferencia → conciliación → cierre.
- **Cero exposición regulatoria** para Rural Pay.
- **Sin onboarding bancario complicado** para los clientes.
- **Sin requerir estatutos** ni documentación sensible.
- **Conciliación semi-automática** que escala (import CSV).
- **Comprobantes archivados** con hash SHA-256 para auditoría.
- **Audit log append-only** de toda acción operativa.

## Lo que falta para producción

- Workflow de notificación al deudor (email cuando se concilia).
- Workflow de recordatorios automáticos (vencimientos próximos).
- OCR opcional sobre comprobantes para extraer monto/fecha/CBU automáticamente.
- Open Banking para conciliación tiempo real.
- Reportes PDF descargables.
- 2FA TOTP para operadores Rural Pay.
- HTTPS + dominio + producción con nginx + Let's Encrypt.
