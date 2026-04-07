---
title: Integración con Bases de Datos Externas
author: Ing. Juan Alberto Sánchez
date: 2026-04-07
version: 1.0
status: Investigación
---

# Integración con Bases de Datos Externas

Guía de integración con proveedores comerciales de bases de datos académicas, enfocado en métodos de autenticación y acceso remoto.

## Panorama de proveedores

| Proveedor | Tipo | IP Auth | SAML 2.0 | OpenURL | Notas |
|-----------|------|---------|----------|---------|-------|
| EBSCO | Multidisciplinar | ✅ | ✅ | ✅ | Más popular en Latinoamérica |
| ProQuest | Multidisciplinar | ✅ | ✅ | ✅ | Incluye tesis, disertaciones |
| Scopus (Elsevier) | Índice de citaciones | ✅ | ✅ | ✅ | Costo alto, muy usado en investigación |
| Web of Science (Clarivate) | Índice de citaciones | ✅ | ✅ | ✅ | Competidor de Scopus |
| Gale (Cengage) | Multidisciplinar | ✅ | ✅ | ✅ | Menos común en Perú |
| Taylor & Francis Online | Revistas | ✅ | ✅ | ✅ | Acceso por metadatos OpenURL |
| SpringerLink | Revistas + Ebooks | ✅ | ✅ | ✅ | Popular en ingeniería |
| JSTOR | Multidisciplinar | ✅ | ✅ | ✅ | Acceso a archive completo |

## Métodos de autenticación soportados

### 1. Autenticación por IP (IP Address Recognition)

**Cómo funciona:**
- La universidad se registra con su(s) IP pública(s)
- El proveedor whitelist esa IP
- Cualquier usuario que accede desde esa IP tiene acceso autorizado
- No requiere login ni SAML

**Ventajas:**
- Transparente para el usuario
- No requiere sincronización de credenciales
- Funciona en redes campus y externas (si hay VPN)

**Desventajas:**
- No vincula uso a usuario específico (analytics débil)
- Requiere VPN para acceso remoto (o proxy)
- Si UPeU cambio IP, hay downtime
- No permite control por rol (alumno vs docente)

**Usado por:**
- EBSCO (opción legacy)
- ProQuest
- Scopus
- Web of Science
- Gale

### 2. SAML 2.0 (Security Assertion Markup Language)

**Cómo funciona:**
1. Usuario accede a portal institutional (Keycloak en UPeU)
2. Portal redirige a base de datos con AuthnRequest SAML
3. Usuario se autentica (password, MFA)
4. Proveedor recibe SAML Response firmado con atributos (email, rol, facultad)
5. Base de datos usa atributos para dar acceso

**Ventajas:**
- Vincula uso a usuario identificado
- Permite analytics por usuario, rol, facultad
- Acceso remoto sin VPN (autenticación + SAML token)
- Standard abierto (no propietario)

**Desventajas:**
- Requiere configuración técnica en ambos extremos
- Algunos proveedores cobran extra por SAML (rare)

**Usado por:**
- EBSCO (opción moderna)
- ProQuest
- Scopus
- Web of Science
- Gale
- SpringerLink
- JSTOR

### 3. OpenURL (Resolver de contexto)

**Cómo funciona:**
1. Usuario en DSpace descubre artículo
2. DSpace genera OpenURL con metadatos: `issn=xxx&title=yyy&author=zzz`
3. Clic en "Find in Library" (resolver local)
4. Resolver (SFX, EZproxy, etc.) mapea a base disponible
5. Redirige a base de datos con query equivalente
6. Acceso se autentica por IP o SAML

**Ventajas:**
- Descubrimiento integrado: usuario busca en DSpace, descubre si está disponible
- Reduce fricción

**Desventajas:**
- Requiere OpenURL resolver local
- EBSCO, ProQuest, etc. ofrecen servidores OpenURL gratis

## Estado actual en universidades peruanas

### CONCYTEC / Acceso Negociado Nacional

CONCYTEC negocia contratos de acceso para universidades peruanas a:
- EBSCO Discovery Service
- Bases especializadas (varía por año)

**Método de autenticación:**
- Históricamente: IP (CONCYTEC negocia un rango de IPs)
- Cambio reciente (2024+): algunos proveedores ofrecen SAML

**Implicación para UPeU:**
- Si UPeU accede vía CONCYTEC → autenticación es por IP
- No se puede cambiar a SAML sin renegociar contrato con CONCYTEC
- Requiere proxy para acceso remoto

### Contratos directos

Si UPeU tiene contratos directos (no vía CONCYTEC):
- Mayor flexibilidad de autenticación
- Posibilidad de negociar SAML
- Costo probablemente más alto

## Estrategia de integración para UPeU

### Paso 1: Auditoría de contratos vigentes

**Preguntas a responder:**

1. ¿Qué bases tienen contratadas?
   - EBSCO Discovery Service (vía CONCYTEC o directo)
   - Scopus (Elsevier — típicamente directo)
   - ProQuest (directo)
   - Web of Science (Clarivate — directo)
   - Springer/Taylor & Francis (directo)
   - Otras

2. ¿Quién es el proveedor contractual?
   - CONCYTEC (negociado nacional)
   - Proveedor directo
   - Distribuidor local (ej: Infotec Perú)

3. ¿Cuál es el método de autenticación actual?
   - IP de UPeU registrada
   - Credenciales por proveedor (usuario + password)
   - SAML (rare, pero posible)

4. ¿El contrato permite cambiar método de autenticación?
   - Si es CONCYTEC → generalmente NO (es costo compartido)
   - Si es directo → probablemente SÍ (negocia con proveedor)

5. ¿Cuál es la vigencia del contrato?
   - Acceso válido hasta cuándo
   - Cuándo renueva

**Responsable:** Bibliotecario de UPeU (Jefe de Biblioteca o Coordinador de Recursos Digitales)

### Paso 2: Clasificar bases por método

**Categoría A — Requieren IP (CONCYTEC)**
- EBSCO Discovery Service vía CONCYTEC
- No se puede cambiar
- Solución: Nginx proxy

**Categoría B — Soportan SAML (directo)**
- Scopus, ProQuest, Web of Science, SpringerLink
- Se puede negociar SAML con proveedor
- Solución: WAYFless URLs con Keycloak

**Categoría C — Flexible (directo + sin SAML actual)**
- Proveedores pequeños, menos conocidos
- Evaluar caso por caso

### Paso 3: Para bases Categoría A (IP)

**Arquitectura:**

```
Usuario (fuera del campus)
    ↓ (https://portal.upeu.edu.pe)
Portal UPeU
    ↓ (requiere Keycloak)
Usuario se autentica
    ↓ (clic en "EBSCO")
Nginx Proxy (IP: 200.x.x.x registrada en EBSCO)
    ↓ (auth_request verifica sesión Keycloak)
Proxy sale hacia EBSCO con IP 200.x.x.x
    ↓
EBSCO reconoce IP → acceso permitido
```

**Configuración Nginx:**

```nginx
server {
    listen 443 ssl http2;
    server_name proxy-bases.upeu.edu.pe;
    ssl_certificate /etc/letsencrypt/live/proxy-bases.upeu.edu.pe/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/proxy-bases.upeu.edu.pe/privkey.pem;

    # verificar autenticación Keycloak
    auth_request /auth;
    auth_request_set $auth_user $upstream_http_x_authenticated_user;
    auth_request_set $auth_status $upstream_status;

    location /auth {
        # internal only
        internal;
        proxy_pass http://keycloak:8080/auth/realms/upeu/protocol/oidc/token/introspect;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header Authorization $http_authorization;
    }

    # EBSCO proxy
    location /ebsco/ {
        auth_request /auth;
        
        proxy_pass https://search.ebscohost.com/;
        proxy_ssl_verify off; # si certificado tiene problemas
        
        # mantener IP saliente (200.x.x.x)
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host search.ebscohost.com;
        proxy_set_header Referer https://proxy-bases.upeu.edu.pe;
        
        # reescribir cookies si es necesario
        proxy_cookie_path / /ebsco/;
    }

    # ProQuest proxy (similar)
    location /proquest/ {
        auth_request /auth;
        proxy_pass https://www.proquest.com/;
        # ... (similar config)
    }
}
```

**Consideraciones:**
- Nginx debe tener IP pública registrada en EBSCO/ProQuest
- Alternativamente, usar proxy transparente en gateway de red
- Logs de Nginx rastrean quién accede a qué

### Paso 4: Para bases Categoría B (SAML)

**Arquitectura:**

```
Usuario en navegador (donde sea)
    ↓ (accede a https://bases.upeu.edu.pe/scopus)
Nginx (redirige a Keycloak)
    ↓
Keycloak login (si no autenticado)
    ↓ (LDAP a AD de UPeU)
Usuario se autentica
    ↓ (Keycloak genera SAML Response)
Keycloak redirige a Scopus con SAML
    ↓
Scopus valida SAML, permite acceso
```

**Pasos para activar SAML con Scopus (Elsevier):**

1. **Contactar soporte de Scopus**
   ```
   support@elsevier.com
   Asunto: "Enable SAML 2.0 authentication for UPeU"
   ```

2. **Proporcionar metadatos de Keycloak:**
   ```
   https://sso.upeu.edu.pe/auth/realms/upeu/protocol/saml/descriptor
   ```

3. **Scopus configura el SP** (Scopus como Service Provider):
   - Valida firma SAML de Keycloak
   - Mapea atributos: email → usuario en Scopus
   - Configuran "Assertion Consumer Service URL" (ACS)

4. **Configurar cliente SAML en Keycloak:**

   ```json
   {
     "clientId": "scopus-saml",
     "name": "Scopus (Elsevier)",
     "protocol": "saml",
     "attributes": {
       "saml.assertion.signature": "true",
       "saml.server.signature": "true",
       "saml.client.signature": "false",
       "saml_assertion_consumer_url_redirect": "https://www.scopus.com/saml/acs"
     },
     "protocolMappers": [
       {
         "name": "email",
         "protocol": "saml",
         "protocolMapper": "saml-user-attribute-mapper",
         "config": {
           "user.attribute": "email",
           "saml.attribute.name": "urn:oid:0.9.2342.19200300.100.1.1"
         }
       }
     ]
   }
   ```

5. **Obtener metadatos de Scopus:**
   ```
   https://www.scopus.com/saml/metadata.xml
   ```

6. **Crear WAYFless URL para portal UPeU:**
   ```html
   <!-- en portal.upeu.edu.pe -->
   <a href="https://sso.upeu.edu.pe/auth/realms/upeu/protocol/saml/clients/scopus-saml">
     Acceder a Scopus
   </a>
   ```

**Pasos similares para ProQuest, Web of Science, SpringerLink**

### Paso 5: Acceso remoto para CONCYTEC (sin SAML)

Si EBSCO via CONCYTEC no permite SAML, alternativa VPN:

**Solución 1: VPN Institucional**
- Usuario se conecta a VPN UPeU
- Obtiene IP del campus
- Accede a EBSCO directamente (sin proxy)
- Simple pero requiere que usuarios instalen cliente VPN

**Solución 2: Proxy transparente**
- Hardware en borde de red (gateway)
- Redirige tráfico HTTPS hacia proveedor
- Conserva IP del campus en conexión saliente
- Más complejo de implementar

**Solución 3: Proxy explícito (Nginx)**
- Usuario accede a portal, autentica en Keycloak
- Portal redirige a proxy Nginx
- Nginx valida sesión y proxea tráfico
- IP saliente = IP de Nginx (registrada en EBSCO)
- Ver configuración en "Paso 3" arriba

## Caso CONCYTEC — Flujo especial

CONCYTEC negocia acceso nacional a:
- EBSCO (EDS — EBSCO Discovery Service)
- Ocasionalmente: Scopus, bases especializadas

**Responsabilidades:**
- CONCYTEC: negocia contrato, obtiene acceso
- Universidad: registra IPs en CONCYTEC
- CONCYTEC: whitelista IPs en EBSCO/proveedores

**Autenticación:**
- IP-based (rango de IPs de la universidad)
- Transparente: usuario accede desde campus, EBSCO lo autoriza

**Para acceso remoto:**
- Universidad debe ofrecer VPN
- O implementar proxy (tema de UPeU resolver)

**Documento relacionado:** `~/obsidian/sciback/08-integraciones/concytec-acceso-nacional.md` (si existe)

## Integración con DSpace — OpenURL

DSpace puede integrarse con servidores OpenURL locales para descubrimiento integrado.

### SFX (ExLibris)

ExLibris ofrece SFX como resolver de contexto:

```xml
<!-- en dspace/modules/discovery/discovery.xml -->
<bean id="org.dspace.discovery.configuration.DiscoveryConfigurationService"
    class="org.dspace.discovery.configuration.DiscoveryConfigurationService">
    
    <property name="openURLBasePath" value="http://upeu.sfx.summon.serialssolutions.com" />
    <property name="openURLLink" value="true" />
</bean>
```

Beneficios:
- En vista de ítem DSpace, aparece botón "Find in Library"
- Si usuario tiene acceso a EBSCO/ProQuest, se redirige
- Si no tiene acceso, muestra alternativas

## Implementación — Orden de prioridad

### Fase 2A (2-4 semanas)

1. Auditoría de contratos UPeU → documento de bases actuales
2. Clasificación: Categoría A (IP) vs B (SAML)
3. Contactar proveedores Categoría B para activar SAML

### Fase 2B (3-5 semanas, paralelo a 2A)

1. Keycloak desplegado y funcionando (Fase 1)
2. Configurar clientes SAML para Categoria B
3. Crear WAYFless URLs en portal

### Fase 2C (1-2 semanas, después de 2B)

1. Implementar proxy Nginx para Categoría A (si aplica)
2. Probar acceso remoto desde red móvil
3. Documentar en manual de usuario

## Checklist para nuevo proveedor

```markdown
Proveedor: ___________________
Contratante: [ ] CONCYTEC [ ] Directo
Método actual: [ ] IP [ ] Credenciales [ ] SAML

[ ] 1. Obtener contacto de soporte técnico
[ ] 2. Revisar contrato: ¿permite cambiar autenticación?
[ ] 3. Solicitar soportar SAML (si es aplicable)
[ ] 4. Obtener metadatos XML de proveedor
[ ] 5. Configurar cliente SAML en Keycloak
[ ] 6. Crear WAYFless URL
[ ] 7. Probar desde IP local (campus)
[ ] 8. Probar desde IP remota (móvil/casa)
[ ] 9. Verificar logs en Keycloak
[ ] 10. Documentar en wiki interna
```

## Errores comunes y solución

### Error: "Invalid Assertion Signature" en Keycloak

**Causa:** Keycloak no confía en el certificado del proveedor

**Solución:**
```bash
# Agregar certificado del proveedor a store de Java
keytool -import -alias scopus-saml -file /path/to/scopus-cert.pem \
  -keystore /opt/keycloak/standalone/configuration/keycloak.jks \
  -storepass password123
```

### Error: Usuario se autentica pero no accede a recurso

**Causa:** Atributos SAML no mapeados correctamente

**Solución:**
1. Verificar en Keycloak: Client → SAML KEYS → Mappers
2. Comparar attribute name en Keycloak vs proveedor
3. Activar debug logs en Keycloak: `DEBUG org.keycloak.saml`

### Error: "RelayState no coincide"

**Causa:** Flujo de redirección SAML interrumpido

**Solución:**
1. Verificar cookie de sesión en navegador
2. Limpiar cookies, volver a intentar
3. Verificar que URL de retorno (ACS) es HTTPS válido

## Referencias

- [SAML 2.0 standard](https://en.wikipedia.org/wiki/SAML_2.0)
- [Keycloak SAML docs](https://www.keycloak.org/docs/latest/server_admin/index.html#saml)
- [EBSCO SAML Integration Guide](https://support.ebsco.com/) (requiere login)
- [Scopus Federation Service](https://service.elsevier.com/)
- [ProQuest SSO Setup](https://www.proquest.com/support)
