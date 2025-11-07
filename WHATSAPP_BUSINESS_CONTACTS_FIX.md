# Fix: WhatsApp Business API - Guardado de Contactos

## Problema Identificado

Al integrar con WhatsApp Business API, los contactos no se estaban guardando en la base de datos, a diferencia de la integración con Baileys (método QR) que funcionaba correctamente.

## Causa Raíz

Se identificaron **3 problemas críticos** en el archivo `whatsapp.business.service.ts`:

### 1. Falta de Validación de `received.contacts`
**Línea 691 (original):**
```typescript
const contactRemoteJid = createJid(received.contacts[0].wa_id);
```

El código accedía directamente a `received.contacts[0].wa_id` sin validar si el array `received.contacts` existía, causando un error cuando el webhook no incluía este campo.

### 2. Sin Fallback al Campo `from`
Según la [documentación oficial de WhatsApp Business API](https://developers.facebook.com/docs/whatsapp/webhooks/), el campo `contacts` puede no estar presente en todos los webhooks, pero el campo `from` en `messages[0]` siempre está disponible.

### 3. Validación Inconsistente de `pushName`
**Línea 390 (original):**
```typescript
if (received.contacts) pushName = received.contacts[0].profile.name;
```

No validaba la existencia de `profile.name`, lo que podía causar errores.

### 4. Falta de Validación de Configuración
No se validaba `DATABASE.SAVE_DATA.CONTACTS` antes de guardar, a diferencia de la implementación en Baileys.

## Solución Implementada

### Cambio 1: Validación de `pushName` (Líneas 390-393)

**Antes:**
```typescript
if (received.contacts) pushName = received.contacts[0].profile.name;
```

**Después:**
```typescript
// Extraer pushName con validación apropiada
if (received.contacts && received.contacts.length > 0 && received.contacts[0].profile?.name) {
  pushName = received.contacts[0].profile.name;
}
```

### Cambio 2: Validación de `contactWaId` con Fallback (Líneas 693-708)

**Antes:**
```typescript
const contactRemoteJid = createJid(received.contacts[0].wa_id);
```

**Después:**
```typescript
// Guardar contacto - FIX: validar received.contacts y usar message.from como fallback
// Según la documentación de WhatsApp Business API, contacts puede no estar presente
// pero message.from siempre está disponible
let contactWaId: string;

if (received.contacts && received.contacts.length > 0 && received.contacts[0].wa_id) {
  contactWaId = received.contacts[0].wa_id;
  this.logger.log(`Usando wa_id de contacts: ${contactWaId}`);
} else {
  // Fallback: usar el campo 'from' del mensaje
  contactWaId = message.from;
  this.logger.log(`Usando message.from como fallback: ${contactWaId}`);
}

const contactRemoteJid = createJid(contactWaId);
this.logger.log(`Guardando contacto con remoteJid: ${contactRemoteJid}`);
```

### Cambio 3: Fallback para `pushName` (Línea 716)

**Antes:**
```typescript
pushName,
```

**Después:**
```typescript
pushName: pushName || contactWaId.split('@')[0],
```

### Cambio 4: Validación de Configuración (Líneas 736-751)

**Antes:**
```typescript
await this.prismaRepository.contact.updateMany({
  where: { remoteJid: contact.remoteJid },
  data: contactRaw,
});

// ...

await this.prismaRepository.contact.create({
  data: contactRaw,
});
```

**Después:**
```typescript
if (this.configService.get<Database>('DATABASE').SAVE_DATA.CONTACTS) {
  await this.prismaRepository.contact.updateMany({
    where: { remoteJid: contact.remoteJid },
    data: contactRaw,
  });
}

// ...

if (this.configService.get<Database>('DATABASE').SAVE_DATA.CONTACTS) {
  await this.prismaRepository.contact.create({
    data: contactRaw,
  });
}
```

## Estructura del Webhook de WhatsApp Business API

Según la documentación oficial, el payload del webhook tiene la siguiente estructura:

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "WHATSAPP_BUSINESS_ACCOUNT_ID",
    "changes": [{
      "value": {
        "messaging_product": "whatsapp",
        "metadata": {
          "display_phone_number": "PHONE_NUMBER",
          "phone_number_id": "PHONE_NUMBER_ID"
        },
        "contacts": [{
          "profile": {
            "name": "NAME"
          },
          "wa_id": "PHONE_NUMBER"
        }],
        "messages": [{
          "from": "PHONE_NUMBER",
          "id": "wamid.ID",
          "timestamp": "TIMESTAMP",
          "text": {
            "body": "MESSAGE_BODY"
          },
          "type": "text"
        }]
      },
      "field": "messages"
    }]
  }]
}
```

**Nota importante:** El campo `contacts` puede no estar presente en todos los webhooks, pero `messages[0].from` siempre está disponible.

## Beneficios de la Solución

1. **Robustez:** Maneja correctamente los casos donde `contacts` no está presente en el webhook
2. **Consistencia:** Sigue el mismo patrón de validación que la implementación de Baileys
3. **Debugging:** Logs adicionales para facilitar el troubleshooting
4. **Configuración:** Respeta la configuración `DATABASE.SAVE_DATA.CONTACTS`
5. **Fallback:** Usa `message.from` cuando `contacts.wa_id` no está disponible

## Testing Recomendado

1. **Reiniciar el servidor** para cargar los cambios:
   ```bash
   # Si estás usando Docker
   docker-compose restart evolution_api
   
   # O si estás ejecutando directamente
   npm run dev:server
   ```

2. **Enviar un mensaje** desde un número de WhatsApp a la instancia de WhatsApp Business API

3. **Verificar en los logs** que aparezcan los mensajes de debugging:
   ```
   [ChannelStartupService] === INICIO GUARDADO DE CONTACTO ===
   [ChannelStartupService] Usando wa_id de contacts: 50762383946
   [ChannelStartupService] Guardando contacto con remoteJid: 50762383946@s.whatsapp.net
   [ChannelStartupService] Contacto nuevo, creando...
   [ChannelStartupService] Contacto creado exitosamente en la base de datos
   ```
   
   O si el contacto ya existe:
   ```
   [ChannelStartupService] === INICIO GUARDADO DE CONTACTO ===
   [ChannelStartupService] Usando wa_id de contacts: 50762383946
   [ChannelStartupService] Guardando contacto con remoteJid: 50762383946@s.whatsapp.net
   [ChannelStartupService] Contacto existente encontrado, actualizando: 50762383946@s.whatsapp.net
   [ChannelStartupService] Contacto actualizado exitosamente en la base de datos
   ```

4. **Verificar en la base de datos** que el contacto se haya guardado correctamente:
   ```sql
   SELECT * FROM contact WHERE instanceId = 'tu-instance-id' ORDER BY createdAt DESC LIMIT 10;
   ```

5. **Verificar webhooks** (si están configurados) que se hayan enviado:
   - `CONTACTS_UPSERT` para contactos nuevos
   - `CONTACTS_UPDATE` para contactos existentes

## Logs de Debugging Agregados

Se agregaron logs detallados en cada paso del proceso de guardado de contactos:

- `=== INICIO GUARDADO DE CONTACTO ===` - Indica que se inició el proceso
- `Usando wa_id de contacts: ...` - Muestra que se usó el campo `wa_id` del payload
- `Usando message.from como fallback: ...` - Muestra que se usó el fallback
- `Guardando contacto con remoteJid: ...` - Muestra el JID formateado
- `Contacto es status@broadcast, omitiendo...` - Indica que se omitió un broadcast
- `Contacto existente encontrado, actualizando: ...` - Indica actualización
- `Contacto nuevo, creando...` - Indica creación
- `Contacto creado/actualizado exitosamente en la base de datos` - Confirma éxito
- `DATABASE.SAVE_DATA.CONTACTS está deshabilitado, no se guardó` - Indica configuración deshabilitada
- `Error al guardar contacto: ...` - Muestra errores si ocurren

## Referencias

- [WhatsApp Business API Webhooks Documentation](https://developers.facebook.com/docs/whatsapp/webhooks/)
- [WhatsApp Cloud API Get Started](https://developers.facebook.com/docs/whatsapp/cloud-api/get-started/)

## Archivos Modificados

- `src/api/integrations/channel/meta/whatsapp.business.service.ts`

## Fecha de Implementación

7 de noviembre de 2024
