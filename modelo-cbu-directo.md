# Modelo operativo: cobranza CBU → CBU directa

Esta es la configuración elegida para Rural Pay: **los fondos no pasan en ningún momento por una cuenta intermedia de Rural Pay ni de la pasarela. Van directo desde el deudor al CBU del cliente final.**

## Implicancias

### 1. Rural Pay NO retiene su comisión del pago

En el modelo split-payment (típico de marketplaces), la pasarela retiene un porcentaje y lo deposita en la cuenta del operador. Con CBU directo eso no se puede hacer automáticamente porque la transferencia bancaria no admite split de fees a un tercero.

**Solución elegida: facturación mensual**

A fin de cada mes, Rural Pay le emite una factura electrónica a cada cliente por el total de fees acumulados de las transacciones del período. La plataforma calcula este monto y expone el detalle en:

```
GET /api/v1/billing/monthly?year=2026&month=5
```

Salida ejemplo:

```json
{
  "period": "2026-05",
  "items": [
    {
      "client_id": 1,
      "client_name": "Cooperativa Agraria del Sur S.A.",
      "fee_percentage": "2.500",
      "gross": "8420000.00",
      "net_to_client": "8209500.00",
      "fee_rural_pay": "210500.00",
      "payments_count": 47
    }
  ],
  "totals": { "fee_rural_pay": "210500.00", "payments_count": 47 }
}
```

El cliente paga esa factura por separado (transferencia o débito acordado).

### 2. Pasarela recomendada: MODO Business + QR Interoperable

| Característica | Por qué |
|---|---|
| Cobertura | Lo escanean apps de ~30 bancos (BBVA, Galicia, Macro, Santander, Nación, etc.) |
| Costo | 0.8% + IVA — el más barato del mercado |
| Acreditación | Inmediata al CBU configurado en onboarding |
| KYB | Onboarding empresarial 1-3 semanas |

### 3. Onboarding de MODO Business (paso a paso para cada cliente)

1. Cada cliente abre cuenta corriente empresarial en un banco socio de MODO.
2. Ingresan a https://modo.com.ar/comercios y solicitan acceso "Business / API".
3. Presentan documentación KYB: CUIT, estatuto, designación de autoridades, comprobante de CBU.
4. MODO les entrega `client_id`, `client_secret` y `merchant_id` (uno por cada cliente).
5. Esas credenciales van en `.env` de **Rural Pay** (no de cada cliente — la plataforma intermedia la creación del cobro pero los fondos van al CBU del merchant).
6. En `.env`: `PAYMENT_GATEWAY=modo`.

### 4. ¿Y mientras se hace el onboarding?

Usamos el adapter **DemoGateway** que simula pagos aprobados para testing del flujo completo.

En `.env`:
```
PAYMENT_GATEWAY=demo
```

La plataforma queda 100% funcional: dashboards, métricas, ledger, facturación mensual — todo se calcula con pagos simulados. Cuando llegan las credenciales de MODO, se cambia una sola línea del `.env` y se reinicia.

### 5. Aspectos regulatorios a considerar

- **Rural Pay no es PSPCP**: como no toca los fondos, no necesita registrarse como Proveedor de Servicios de Pago de Cobranza de Pagos ante el BCRA.
- **Facturación**: Rural Pay factura un "servicio de gestión de cobranzas" (no procesa pagos legalmente). Es una factura A o B según condición fiscal del cliente.
- **Confidencialidad**: la plataforma maneja datos personales de deudores. Aplica Ley 25.326 (Protección de Datos Personales). Tener política de privacidad y registro ante la AAIP.
- **Audit log**: la plataforma ya registra toda acción en `audit_logs` para trazabilidad.

### 6. Diagrama del flujo

```
Deudor ──pago──> [MODO/QR] ──CBU directo──> Cuenta corriente Cliente
                    │
                    └── webhook ──> Rural Pay actualiza estado, registra fee
                                              │
                                              └── Fin de mes ──> Factura
                                                              al cliente
                                                                ↓
                                                  Cliente paga fee a Rural Pay
```

### 7. Cambiar de modelo en el futuro

Si en algún momento se quiere mover a un modelo de marketplace con split automático (Rural Pay retiene fee del mismo pago), el camino sería:

- Migrar el flujo a Mercado Pago marketplace con OAuth (`application_fee`).
- Cambiar `PAYMENT_GATEWAY=mercadopago` en `.env`.
- Cada cliente se "conecta" desde la app vía un endpoint OAuth de MP.
- El código del adapter ya está en `app/integrations/mercadopago.py`, solo hay que agregar el flujo OAuth.

Esto requeriría que Rural Pay se registre como "plataforma" en MP, KYB de Anthropic… digo, de MP.
