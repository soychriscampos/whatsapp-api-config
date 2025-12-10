# Integración completa de WhatsApp Cloud API con número propio (Colegio REAL de Escuinapa)

## Descripción general

Este documento describe el proceso completo que seguí para conectar un número propio (una eSIM Telcel) con la WhatsApp Cloud API, crear la infraestructura base en Vercel, vincularla con mi dominio personalizado, y lograr la verificación, registro y conexión exitosa del número.

El objetivo fue dejar todo documentado para replicar fácilmente el flujo en futuros proyectos sin repetir errores.

---

## Requisitos previos

- Número de teléfono nuevo (en mi caso, eSIM Telcel).
- Un dominio personalizado (en mi caso, Namecheap -> xxxx.app).
- Cuenta en Meta Developers.
- Acceso a Meta Business Manager.
- Tarjeta configurada en el método de pago del Business Manager.
- Acceso a Vercel y GitHub.

---

## 1. Compra del número y dominio

1. Compré una eSIM nueva en Telcel.
2. Ya tenía un dominio en Namecheap, configurado con DNS personalizado:
   ```
   xxxx.xxxxxx.app
   wa.xxxxxx.app
   ```
   Ambos apuntan a Vercel mediante CNAME.

---

## 2. Creación de la aplicación en Meta Developers

1. Entré a developers.facebook.com 
   -> Create App -> Business App.
2. Le asigné un nombre identificable (por ejemplo, `adminname`).
3. Dentro del panel de la app, agregué el producto WhatsApp.
4. En la sección API Setup, registré el número de teléfono (la eSIM recién adquirida).
5. Elegí la verificación vía SMS.

En este punto, Meta comenzó a mostrar el número con:
```
Status: Pending
Once the display name is approved, download the certificate to connect this phone number.
```

---

## 3. Configuración en Meta Business Manager

> Esta fue la parte más confusa, pero esencial para poder generar un token permanente.

1. Fui a Meta Business Manager -> Business Settings -> Accounts -> Apps.  
   Allí agregué la aplicación que había creado en Developers.
2. Luego, en Users -> People, agregué un usuario Admin (mi cuenta principal).
3. Le otorgué permisos completos a la App y a la cuenta de WhatsApp.
4. En System Users, creé un nuevo System User (nombre: `sample-name-bot`).
5. Asigné a ese usuario los permisos de:
   - `whatsapp_business_messaging`
   - `whatsapp_business_management`
6. En el System User, generé un Permanent Token (token largo).
7. Seleccioné las apps que usaría (mi app creada en Developers) y los permisos de WhatsApp.

*Este token es el que se usó después para el registro final vía* `curl`.

---

## 4. Variables de entorno (.env)

En el entorno local creé un archivo `.env` (ignorado por git):

```
WHATSAPP_VERIFY_TOKEN=<token_simple_de_verificación>
META_APP_SECRET=<app_secret_de_meta>
META_PERMANENT_TOKEN=<token_permanente_del_system_user>
WHATSAPP_PHONE_NUMBER_ID=<wa_pn_id>
```

Luego, las mismas variables se agregaron al proyecto en Vercel -> Settings -> Environment Variables.

---

## 5. Configuración del proyecto

1. Creé un nuevo repositorio en GitHub (`wa-webhook`).
2. Inicialicé el entorno local y subí los archivos base.
3. Vinculé el repositorio a un **nuevo proyecto en Vercel**.
4. Configuré el subdominio:
   ```
   wa.xxxxxx.app
   ```
5. Subí los archivos:
   ```
   api/webhook.py
   api/send.py
   requirements.txt
   vercel.json
   ```

---

## 6. Webhook

En `api/webhook.py` definí el endpoint con verificación por token:

```python
import os
from flask import Flask, request, abort, Response
app = Flask(__name__)
VERIFY_TOKEN = os.environ.get("WHATSAPP_VERIFY_TOKEN")

@app.route("/api/webhook", methods=["GET"])
def verify():
    mode = request.args.get("hub.mode")
    token = request.args.get("hub.verify_token")
    challenge = request.args.get("hub.challenge")
    if mode == "subscribe" and token == VERIFY_TOKEN:
        return Response(challenge, status=200)
    return abort(403)

@app.route("/api/webhook", methods=["POST"])
def receive():
    data = request.get_json()
    print("WA Webhook:", data)
    return Response("EVENT_RECEIVED", status=200)
```

Luego verifiqué manualmente:
```
https://wa.xxxxxx.app/api/webhook?hub.mode=subscribe&hub.verify_token=<TOKEN>&hub.challenge=1234
```
-> devolvió `1234`, lo que confirmó la conexión.

*links deshabilitados*

---

## 7. Configuración del webhook en Meta Developers

1. En la App -> WhatsApp -> Configuration -> Webhooks
2. Producto: WhatsApp
3. Callback URL:
   ```
   https://wa.xxxxxx.app/api/webhook
   ```
4. Verify Token: (el mismo de `.env`)
5. Desactivar “Attach client certificate”.
6. Click en Verify and Save.
7. Suscribirse a:
   - `messages`
   - `message_template_status_update`

_Estado: “Webhook verified successfully”._

---

## 8. Método de pago

Meta no activa los números reales sin un método de pago configurado.  
Ingresé a Business Settings -> Pagos y agregué una tarjeta.

---

## 9. Registro del número (proceso real)

Una vez aprobado el display name, el estado seguía “Pending”.  
Entonces ejecuté los comandos siguientes:

### Intento 1 — Sin PIN (falló)
```bash
curl -X POST "https://graph.facebook.com/v21.0/WHATSAPP_PHONE_NUMBER_ID/register"   -H "Authorization: Bearer <MI_TOKEN>"   -H "Content-Type: application/json"   -d '{ "messaging_product": "whatsapp" }'
```
Resultado: **solicitaba un PIN**.

### Intento 2 — Con PIN (éxito)
```bash
curl -X POST "https://graph.facebook.com/v21.0/WHATSAPP_PHONE_NUMBER_ID/register"   -H "Authorization: Bearer <MI_TOKEN>"   -H "Content-Type: application/json"   -d '{ "messaging_product": "whatsapp", "pin": "000000" }'
```

Resultado:  
```json
{"success": true}
```
El número quedó **verificado y registrado**.

---

## 10. Webhook activo y mensaje de prueba

Para verificar funcionamiento completo, creé el endpoint `/api/send.py` y un template (en revisión).

Intenté enviar un mensaje con `hello_world`, pero apareció el error:
```
"hello_world templates can only be sent from the Public Test Numbers"
```
Se resolvió creando **una plantilla propia** (`prueba_saludo`) en WhatsApp Manager.

---

## 11. Errores comunes que enfrenté

| Error | Causa | Solución |
|-------|--------|----------|
| `(#133010) Account not registered` | Faltaba ejecutar `/register` | Ejecutar comando con token correcto |
| `(#131047) PIN required` | Número reciclado con 2FA heredado | Registrar con `"pin": "000000"` |
| “Pending” prolongado | Display name aprobado pero no registrado | Registrar manualmente con curl |
| `hello_world templates…` | Plantilla de test restringida | Crear plantilla propia |
| Webhook Forbidden | GET sin parámetros | Verificar `hub.verify_token` |

---

## 12. Estado final

- Número **registrado correctamente**
- Webhook verificado (`/api/webhook`)
- Token permanente activo
- Mensajes salientes disponibles (con plantilla propia)
- Integración lista para automatizar recordatorios

---

## 13. Estructura del proyecto

```
wa-webhook/
├── api/
│   ├── webhook.py
│   ├── send.py
├── requirements.txt
├── vercel.json
├── .env (local, gitignored)
└── README.md
└── public.md
```

---

## 14. Duración y aprendizajes

Duración total: **~48 horas**  
Principales obstáculos superados:
- Diferencia entre **verificación** y **registro real** del número.
- Números reciclados con **PIN heredado** (solución: `"000000"`).
- Display name “Pending” aunque ya aprobado.
- Limitación de `hello_world` para números de prueba.
- Importancia de tener el **método de pago activo**.

---

## 15. Referencias rápidas (curl)

Verificar número:
```bash
curl -X GET "https://graph.facebook.com/v21.0/<WABA_ID>/phone_numbers"   -H "Authorization: Bearer <TOKEN>"
```

Registrar número:
```bash
curl -X POST "https://graph.facebook.com/v21.0/WHATSAPP_PHONE_NUMBER_ID/register"   -H "Authorization: Bearer <TOKEN>"   -H "Content-Type: application/json"   -d '{ "messaging_product": "whatsapp", "pin": "000000" }'
```

Enviar mensaje:
```bash
curl -X POST "https://graph.facebook.com/v21.0/WHATSAPP_PHONE_NUMBER_ID/messages"   -H "Authorization: Bearer <TOKEN>"   -H "Content-Type: application/json"   -d '{
    "messaging_product":"whatsapp",
    "to":"52166XXXXXXXX",
    "type":"text",
    "text":{"body":"Hola — prueba desde XXXXX"}
  }'
```

---

**Autor:** Christian Campos  
**Proyecto:** XXXXX – Automatización de notificaciones vía WhatsApp Cloud API  
**Fecha:** Noviembre 2025  
