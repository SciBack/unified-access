---
title: Roadmap de Implementación — Unified Access UPeU
author: Ing. Juan Alberto Sánchez
date: 2026-04-07
version: 1.0
status: Plan
---

# Roadmap de Implementación

Plan de despliegue en 4 fases, 8-12 meses.

## Fase 1: Fundación — SSO Keycloak + Apps internas

**Duración:** 6-8 semanas  
**Objetivo:** Usuarios logueados con SSO en DSpace, OJS, Koha, Moodle sin duplicar credenciales  
**Deliverables:** Keycloak desplegado, 4 apps integradas, portal básico

### Subtarea 1.1: Infraestructura — Keycloak en Docker

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 1-2 semanas

- [ ] Provision servidor Ubuntu 22.04 en VMware de UPeU
  - CPU: 4 cores
  - RAM: 16GB
  - Disk: 100GB (para base de datos y logs)
  - Hostname: `sso.upeu.edu.pe`
  - IP privada: 192.168.x.x
  - IP pública: X.X.X.X (Elastic IP si es AWS)

- [ ] Instalar Docker + Docker Compose
  ```bash
  curl -fsSL https://get.docker.com | bash
  sudo usermod -aG docker ubuntu
  ```

- [ ] Crear stack Docker Compose con:
  - Keycloak 24.0.x (última estable)
  - PostgreSQL 15 (base de datos de Keycloak)
  - Nginx (reverse proxy SSL)
  - Certificado SSL Let's Encrypt

- [ ] Docker Compose file: `docker-compose.yml`
  ```yaml
  version: '3.9'
  services:
    postgres:
      image: postgres:15-alpine
      environment:
        POSTGRES_DB: keycloak
        POSTGRES_USER: keycloak
        POSTGRES_PASSWORD: ${DB_PASSWORD}
      volumes:
        - postgres_data:/var/lib/postgresql/data

    keycloak:
      image: keycloak/keycloak:24.0.0
      environment:
        KC_DB: postgres
        KC_DB_URL: jdbc:postgresql://postgres/keycloak
        KC_DB_USERNAME: keycloak
        KC_DB_PASSWORD: ${DB_PASSWORD}
        KEYCLOAK_ADMIN: admin
        KEYCLOAK_ADMIN_PASSWORD: ${ADMIN_PASSWORD}
      ports:
        - "8080:8080"
      depends_on:
        - postgres

    nginx:
      image: nginx:alpine
      ports:
        - "443:443"
        - "80:80"
      volumes:
        - ./nginx.conf:/etc/nginx/nginx.conf
        - /etc/letsencrypt:/etc/letsencrypt

  volumes:
    postgres_data:
  ```

- [ ] Obtener certificado SSL
  ```bash
  sudo certbot certonly --standalone -d sso.upeu.edu.pe
  ```

- [ ] Configurar Nginx como reverse proxy
  - Escuchar en :443 HTTPS
  - Proxear a Keycloak :8080
  - Reescribir headers

- [ ] Test: Acceder a `https://sso.upeu.edu.pe` → admin console

- [ ] Documentar credenciales en `~/.secrets/keycloak.env`

### Subtask 1.2: Conectar Keycloak con AD de UPeU

**Responsable:** Ing. Juan Alberto Sánchez + Admin AD de UPeU  
**Duración:** 1-2 semanas

**Requisitos previos:**
- IP de Keycloak debe alcanzar AD (puerto 389/636)
- Credenciales de lectura AD (usuario de servicio, ej: `ldapbind@upeu.edu.pe`)

**Tareas:**

- [ ] Recopilar info de AD:
  ```
  Domain: upeu.edu.pe o similar
  LDAP Server: 192.168.x.x:389
  Base DN: dc=upeu,dc=edu,dc=pe
  User DN template: uid={0},ou=students,dc=upeu,dc=edu,dc=pe
  ```

- [ ] En Keycloak Admin console:
  - Crear realm `upeu` (si no existe)
  - User Federation → LDAP
  - Configurar:
    - Connection URL: `ldap://192.168.x.x:389`
    - Bind DN: `ldapbind@upeu.edu.pe`
    - Bind Credential: password
    - Base DN: `dc=upeu,dc=edu,dc=pe`

- [ ] Mapear atributos LDAP → Keycloak
  ```
  LDAP cn         → Keycloak name
  LDAP mail       → Keycloak email
  LDAP givenName  → Keycloak givenName
  LDAP surname    → Keycloak surname
  LDAP department → Keycloak department (custom)
  ```

- [ ] Test: Ir a Keycloak login page
  - Ingresar usuario de AD de prueba
  - Verificar que se autentica correctamente

- [ ] Verificar en Keycloak que usuario fue sincronizado
  - Users → buscar usuario de prueba
  - Ver atributos importados de AD

### Subtask 1.3: Integrar DSpace 7.6.6 con Keycloak (SAML)

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 1-2 semanas

**Requisitos:**
- DSpace 7.6.6 ya desplegado en UPeU
- o crear instancia de prueba en Docker

**Pasos:**

- [ ] En Keycloak, crear cliente SAML para DSpace
  ```json
  {
    "clientId": "dspace-saml",
    "name": "DSpace 7.6.6 — UPeU",
    "protocol": "saml",
    "attributes": {
      "saml.assertion.signature": "true",
      "saml.server.signature": "true",
      "saml_single_logout_service_url_redirect": "https://dspace.upeu.edu.pe/logout"
    }
  }
  ```

- [ ] Obtener metadatos XML de Keycloak
  ```
  https://sso.upeu.edu.pe/auth/realms/upeu/protocol/saml/descriptor
  ```

- [ ] En DSpace, editar `dspace/config/modules/authentication-saml.cfg`
  ```properties
  saml.enabled = true
  saml.idp.metadata.url = https://sso.upeu.edu.pe/auth/realms/upeu/protocol/saml/descriptor
  saml.idp.entity_id = https://sso.upeu.edu.pe/auth/realms/upeu
  
  # Mapeo de atributos
  saml.assertion.consumer.service.index = 0
  saml.user.eppn = urn:oid:1.3.6.1.4.1.5923.1.1.1.7
  saml.user.email = urn:oid:0.9.2342.19200300.100.1.1
  saml.user.givenname = urn:oid:2.5.4.42
  saml.user.surname = urn:oid:2.5.4.4
  ```

- [ ] Reiniciar DSpace
  ```bash
  docker-compose restart dspace
  ```

- [ ] Test: Acceder a `https://dspace.upeu.edu.pe/login`
  - Hacer clic en "SAML Login"
  - Debería redirigir a Keycloak
  - Ingresar usuario de AD
  - Redirigir de vuelta a DSpace autenticado

- [ ] Verificar en DSpace que usuario fue creado
  ```bash
  dspace user -l | grep usuario_prueba
  ```

### Subtask 1.4: Integrar OJS 3.x con Keycloak (OIDC)

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 1-2 semanas

**Requisitos:**
- OJS 3.x instalado en UPeU
- o instancia de prueba en Docker

**Pasos:**

- [ ] En Keycloak, crear cliente OIDC para OJS
  ```json
  {
    "clientId": "ojs",
    "name": "OJS 3.x — UPeU",
    "protocol": "openid-connect",
    "clientAuthenticatorType": "client-secret",
    "standardFlowEnabled": true,
    "redirectUris": ["https://ojs.upeu.edu.pe/index.php/index/user/oauthCallback"],
    "webOrigins": ["https://ojs.upeu.edu.pe"]
  }
  ```

- [ ] Obtener OIDC credentials de Keycloak
  - Client ID: `ojs`
  - Client Secret: (generar en Keycloak)
  - Discovery URL: `https://sso.upeu.edu.pe/auth/realms/upeu/.well-known/openid-configuration`

- [ ] En OJS, editar `config.inc.php`
  ```ini
  [oidc]
  enabled = On
  client_id = ojs
  client_secret = <SECRET_FROM_KEYCLOAK>
  discovery_url = https://sso.upeu.edu.pe/auth/realms/upeu/.well-known/openid-configuration
  
  [openid]
  provider = keycloak
  client_id = ojs
  client_secret = <SECRET>
  ```

- [ ] Instalar plugin PKP OIDC (si no viene incluido)
  ```bash
  cd ojs/plugins/generic
  git clone https://github.com/pkp/pkpOidcPlugin.git
  cd pkpOidcPlugin && npm install && npm run build
  ```

- [ ] Test: Acceder a OJS
  - Login → OIDC option
  - Debería redirigir a Keycloak
  - Ingresar usuario de AD
  - Redirigir a OJS autenticado

### Subtask 1.5: Integrar Koha con Keycloak (SAML via plugin)

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 1-2 semanas

**Requisitos:**
- Koha instalado en UPeU
- o instancia de prueba en Docker

**Pasos:**

- [ ] En Keycloak, crear cliente SAML para Koha
  ```json
  {
    "clientId": "koha-saml",
    "name": "Koha — UPeU",
    "protocol": "saml"
  }
  ```

- [ ] En Koha, editar `etc/koha-conf.xml`
  ```xml
  <useshibboleth>1</useshibboleth>
  <shibboleth>
    <idp_entity_id>https://sso.upeu.edu.pe/auth/realms/upeu</idp_entity_id>
    <idp_metadata_url>https://sso.upeu.edu.pe/auth/realms/upeu/protocol/saml/descriptor</idp_metadata_url>
  </shibboleth>
  ```

- [ ] Configurar Apache para Shibboleth
  ```apache
  <Location /Shibboleth.sso>
    Order allow,deny
    Allow from all
  </Location>

  <Location ~ "^/cgi-bin/koha/.*">
    AuthType shibboleth
    Require shibboleth
    ShibRequestSetting requireSession true
  </Location>
  ```

- [ ] Test: Acceder a Koha
  - Debería redirigir a Keycloak
  - Ingresar usuario
  - Redirigir a Koha autenticado

### Subtask 1.6: Integrar Moodle con Keycloak (SAML)

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 1-2 semanas

**Requisitos:**
- Moodle instalado en UPeU
- o instancia de prueba en Docker

**Pasos:**

- [ ] En Keycloak, crear cliente SAML para Moodle
  ```json
  {
    "clientId": "moodle-saml",
    "name": "Moodle — UPeU"
  }
  ```

- [ ] En Moodle Admin panel:
  - Site Administration → Plugins → Authentication → SAML 2.0
  - Configurar:
    - IDP metadata URL: `https://sso.upeu.edu.pe/auth/realms/upeu/protocol/saml/descriptor`
    - Service provider name: `https://moodle.upeu.edu.pe/`

- [ ] Test: Acceder a Moodle
  - Hacer clic en login
  - Debería haber opción SAML
  - Redirigir a Keycloak
  - Ingresar usuario
  - Redirigir a Moodle autenticado

### Subtask 1.7: Portal básico de recursos

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 1-2 semanas

Crear portal web simple (`portal.upeu.edu.pe`) que muestre:
- Información del usuario logueado
- Lista de aplicaciones disponibles (DSpace, OJS, Koha, Moodle)
- Enlaces directos a cada app

**Tecnología:** HTML estático + JavaScript para obtener perfil de Keycloak

```html
<!-- portal/index.html -->
<!DOCTYPE html>
<html>
<head>
  <title>Portal de Recursos — UPeU</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <nav>
    <div id="user-info">
      Bienvenido, <span id="user-name"></span>
      <a href="/logout">Salir</a>
    </div>
  </nav>

  <main>
    <h1>Recursos Digitales</h1>
    <div class="apps-grid">
      <a href="https://dspace.upeu.edu.pe" class="app-card">
        <h3>DSpace</h3>
        <p>Repositorio institucional</p>
      </a>
      <a href="https://ojs.upeu.edu.pe" class="app-card">
        <h3>OJS</h3>
        <p>Revista institucional</p>
      </a>
      <a href="https://koha.upeu.edu.pe" class="app-card">
        <h3>Koha</h3>
        <p>Biblioteca digital</p>
      </a>
      <a href="https://moodle.upeu.edu.pe" class="app-card">
        <h3>Moodle</h3>
        <p>Plataforma educativa</p>
      </a>
    </div>
  </main>

  <script src="keycloak.js"></script>
  <script>
    // Obtener perfil del usuario
    fetch('/auth/userinfo', {
      headers: { 'Authorization': 'Bearer ' + keycloak.token }
    })
    .then(r => r.json())
    .then(user => {
      document.getElementById('user-name').textContent = user.given_name;
    });
  </script>
</body>
</html>
```

## Fase 2: Governance — MidPoint + Vigencias

**Duración:** 6-10 semanas (puede solaparse parcialmente con Fase 1)  
**Objetivo:** Sincronización automática de roles y vigencias desde sistema de matrícula  
**Deliverables:** MidPoint desplegado, provisioning automático, portal de autogestión

### Subtask 2.1: Investigación — Sistema de matrícula UPeU

**Responsable:** Bibliotecario de UPeU + IT  
**Duración:** 1-2 semanas

- [ ] Recopilar info sobre sistema de matrícula:
  - ¿En qué sistema está (BANNER, ARGOS, otro)?
  - ¿Tiene API REST o solo exporta CSV?
  - ¿Qué atributos expone?
    - Usuario (matrícula)
    - Nombre
    - Facultad
    - Carrera
    - Nivel (pregrado/posgrado)
    - Estado (activo/egresado/suspendido)
    - Fecha de matrícula
    - Fecha de fin de vigencia (semestre actual)
  - ¿Frecuencia de actualización (diaria, semanal, mensual)?

- [ ] Crear documento: `docs/investigacion-pendiente.md` → sección "Matrícula"

### Subtask 2.2: Desplegar MidPoint en Docker

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 2-3 semanas

- [ ] Provision servidor Ubuntu 22.04
  - CPU: 4 cores
  - RAM: 16GB
  - Disk: 100GB
  - Hostname: `governance.upeu.edu.pe`

- [ ] Docker Compose con:
  - MidPoint 4.9.x (latest)
  - PostgreSQL 15 (BD de MidPoint)
  - Nginx reverse proxy SSL

- [ ] Test: Acceder a MidPoint
  - `https://governance.upeu.edu.pe/midpoint`
  - Login inicial (admin / password)
  - Cambiar contraseña

- [ ] Documentar credenciales en `~/.secrets/midpoint.env`

### Subtask 2.3: Conectar MidPoint con AD

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 1-2 semanas

- [ ] En MidPoint:
  - Crear Resource (connector LDAP)
  - Connection URL: `ldap://192.168.x.x:389`
  - Base DN: `dc=upeu,dc=edu,dc=pe`
  - Bind credentials: ldapbind@upeu.edu.pe

- [ ] Crear Object Type Mapping
  - LDAP objectClass `inetOrgPerson` → Midpoint `UserType`
  - Mapear atributos

- [ ] Test: Recon (reconciliation)
  - Ejecutar recon de usuarios de AD
  - Verificar que MidPoint importa usuarios

### Subtask 2.4: Conectar MidPoint con sistema de matrícula

**Responsable:** Ing. Juan Alberto Sánchez + Admin Matrícula  
**Duración:** 2-3 semanas

**Dos escenarios posibles:**

**Escenario A: Sistema de matrícula tiene API REST**
- [ ] Crear Resource con connector REST en MidPoint
- [ ] Mapear campos JSON de API a atributos de usuario

**Escenario B: Sistema de matrícula exporta CSV**
- [ ] Crear script en Python que:
  - Descarga CSV de matrícula (o accede a base de datos via ODBC)
  - Parsea y normaliza
  - Carga en tabla `matricula` en PostgreSQL de MidPoint
  - Se ejecuta diariamente vía cron

- [ ] En MidPoint:
  - Crear Resource con connector JDBC
  - Connection string a PostgreSQL
  - Query: `SELECT * FROM matricula`

### Subtask 2.5: Definir roles derivados

**Responsable:** Ing. Juan Alberto Sánchez + Bibliotecario  
**Duración:** 1-2 semanas

En MidPoint, crear roles derivados basados en datos de matrícula:

```xml
<!-- Rol "alumno" -->
<archetypeOid>alumno</archetypeOid>
<inducement>
  <name>es alumno activo</name>
  <focusType>UserType</focusType>
  <condition>
    <script>
      <code>
        user.findProperty(qname('http://upeu.edu.pe/midpoint/schema', 'estado')) == 'ACTIVO'
        AND user.findProperty(qname('http://upeu.edu.pe/midpoint/schema', 'tipo_rol')) == 'ALUMNO'
      </code>
    </script>
  </condition>
  <assignment>
    <!-- asignar este rol -->
  </assignment>
</inducement>

<!-- Rol "egresado" -->
<archetypeOid>egresado</archetypeOid>
<inducement>
  <condition>
    <script>
      <code>
        java.time.LocalDate.now() &gt; user.findProperty(qname('http://upeu.edu.pe/midpoint/schema', 'fecha_fin_vigencia'))
      </code>
    </script>
  </condition>
</inducement>

<!-- Rol derivado "puede_acceder_a_EBSCO" (solo alumno/docente activos) -->
<archetypeOid>ebsco_access</archetypeOid>
<inducement>
  <condition>
    <script>
      <code>
        (user.hasRole('alumno') OR user.hasRole('docente'))
        AND user.getStatus() == 'ENABLED'
      </code>
    </script>
  </condition>
</inducement>
```

### Subtask 2.6: Provisioning automático a Keycloak

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 2-3 semanas

- [ ] En MidPoint:
  - Crear Resource (connector OIDC) hacia Keycloak
  - API endpoint: `https://sso.upeu.edu.pe/auth/admin/realms/upeu/users`
  - Credenciales: service account en Keycloak

- [ ] Crear Mapping:
  - MidPoint user → Keycloak user
  - Atributos sincronizados:
    - uid → username
    - email
    - givenName
    - surname
    - facultad → custom attribute
    - carrera → custom attribute
    - nivel → custom attribute
    - estado → active/inactive

- [ ] Crear Policy:
  - Sync policy: `on demand` (cuando MidPoint cambia)
  - Frecuencia: cada 15 minutos

- [ ] Test:
  - Cambiar facultad de usuario en matrícula
  - Ejecutar recon en MidPoint
  - Verificar que Keycloak se actualiza

### Subtask 2.7: Portal de autogestión para bibliotecarios

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 2-3 semanas

Crear portal para que bibliotecarios puedan:
- Buscar usuario
- Ver atributos (facultad, carrera, vigencia)
- Ver histórico de cambios
- Forzar sincronización manual
- Ver estado de provisioning

**Tecnología:** MidPoint tiene portal de autogestión nativo

- [ ] Habilitar self-service portal en MidPoint
- [ ] Configurar permisos:
  - Bibliotecarios: pueden buscar usuarios pero no modificar
  - Admin: acceso total

- [ ] URL: `https://governance.upeu.edu.pe/midpoint/self`

## Fase 3: Acceso a bases de datos externas

**Duración:** 4-6 semanas  
**Objetivo:** Integración con EBSCO, Scopus, ProQuest, Web of Science  
**Deliverables:** WAYFless URLs configuradas, proxy para bases IP-only

### Subtask 3.1: Auditoría de contratos UPeU

**Responsable:** Bibliotecario UPeU + abogado  
**Duración:** 1-2 semanas

- [ ] Identificar bases de datos contratadas
- [ ] Documentar:
  - Proveedor
  - Método de autenticación actual (IP, credenciales, SAML)
  - Contratante (CONCYTEC directo)
  - Vigencia del contrato
  - Costo anual (si es directo)

- [ ] Documento: `docs/bases-externas.md` → sección "Estado actual en UPeU"

### Subtask 3.2: Activar SAML con proveedores (EBSCO, Scopus, etc.)

**Responsable:** Ing. Juan Alberto Sánchez + Bibliotecario  
**Duración:** 2-4 semanas

Para cada base que soporte SAML:

- [ ] Abrir ticket con soporte del proveedor
  - Solicitar: "Enable SAML 2.0 authentication"
  - Proporcionar metadata de Keycloak

- [ ] Una vez que proveedor configura su SP:
  - Obtener metadata XML del proveedor
  - En Keycloak: crear cliente SAML
  - Configurar ACS (Assertion Consumer Service) URL

- [ ] Crear WAYFless URL en portal UPeU
  ```html
  <a href="https://sso.upeu.edu.pe/auth/realms/upeu/protocol/saml/clients/scopus">
    Acceder a Scopus
  </a>
  ```

- [ ] Test: acceder desde IP remota (4G, WiFi público)

### Subtask 3.3: Implementar proxy Nginx para bases solo-IP

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 1-2 semanas

Para bases que solo aceptan IP:

- [ ] Desplegar Nginx proxy en servidor con IP pública registrada
  - Hostname: `proxy-bases.upeu.edu.pe`
  - IP: registrada en EBSCO/ProQuest/etc.

- [ ] Configurar Nginx:
  - auth_request → verificar sesión Keycloak
  - proxy_pass → base de datos
  - Reescribir cookies si es necesario

- [ ] Test: acceder a proxy desde IP remota
  - Usuario se autentica en Keycloak
  - Proxy redirige a base
  - Base autoriza por IP

### Subtask 3.4: Documentar para usuarios finales

**Responsable:** Bibliotecario UPeU  
**Duración:** 1 semana

- [ ] Crear guía "Acceso a bases de datos remotas"
  - Paso a paso con capturas
  - Troubleshooting común
  - Contacto de soporte

## Fase 4: Analítica

**Duración:** 6-8 semanas  
**Objetivo:** Visibilidad de uso de recursos por facultad, carrera, período  
**Deliverables:** Dashboard Metabase, reportes diarios, alertas

### Subtask 4.1: Desplegar Metabase en Docker

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 1-2 semanas

- [ ] Docker Compose con:
  - Metabase (latest)
  - PostgreSQL (BD de Metabase)
  - Nginx reverse proxy

- [ ] URL: `https://analytics.upeu.edu.pe`

### Subtask 4.2: Crear schema de analítica en PostgreSQL

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 1-2 semanas

- [ ] Crear BD `analytics` con tabla `eventos`
  ```sql
  CREATE TABLE analytics.eventos (
    id SERIAL PRIMARY KEY,
    fecha DATE,
    usuario VARCHAR(255),
    facultad VARCHAR(255),
    carrera VARCHAR(255),
    nivel VARCHAR(50),
    tipo_evento VARCHAR(50),
    recurso VARCHAR(255),
    accion VARCHAR(255),
    timestamp TIMESTAMP,
    ip_cliente INET
  );
  ```

### Subtask 4.3: ETL — Colector de eventos

**Responsable:** Ing. Juan Alberto Sánchez  
**Duración:** 2-3 semanas

Crear script Python que corra cada noche (cron):

```python
# extract_analytics.py
import requests
import psycopg2
from datetime import datetime, timedelta

# 1. Keycloak events
keycloak_events = requests.get(
    'https://sso.upeu.edu.pe/auth/admin/realms/upeu/events',
    headers={'Authorization': f'Bearer {admin_token}'}
).json()

# 2. DSpace stats API
dspace_stats = requests.get(
    'https://dspace.upeu.edu.pe/server/api/statistics/searches',
    params={'limit': 1000}
).json()

# 3. Normalizar y cargar en PostgreSQL
for event in keycloak_events:
    # extraer usuario, timestamp, tipo
    # buscar atributos en MidPoint
    # insertar en analytics.eventos

conn = psycopg2.connect(...)
cur = conn.cursor()
cur.execute('INSERT INTO analytics.eventos (...) VALUES (...)')
conn.commit()
```

- [ ] Crear servicio systemd o cron job
  ```bash
  0 2 * * * /usr/local/bin/extract_analytics.py >> /var/log/analytics.log
  ```

### Subtask 4.4: Dashboards Metabase

**Responsable:** Ing. Juan Alberto Sánchez + Bibliotecario  
**Duración:** 2-3 semanas

Crear dashboards:

1. **Resumen mensual**
   - Total de logins
   - Total de descargas
   - Bases más usadas
   - Facultades con mayor uso

2. **Desglose por facultad**
   - Top 10 facultades por uso
   - Recursos especializados por facultad

3. **Por recurso**
   - Uso de EBSCO vs Scopus vs ProQuest
   - Tendencia mensual
   - Alertas si uso es anómalo

4. **ROI de licencias**
   - Costo por descargas vs presupuesto
   - Propuesta de renovación

## Hitos y milestones

| Fecha | Hito | Responsable |
|-------|------|-------------|
| 2026-05-15 | Fase 1.1-1.2 completadas (Keycloak + AD) | Juan Alberto |
| 2026-06-01 | Fase 1.3-1.6 completadas (Apps SSO) | Juan Alberto |
| 2026-06-15 | Fase 1.7 (Portal) | Juan Alberto |
| 2026-07-01 | Fase 2.1-2.3 (MidPoint inicial) | Juan Alberto |
| 2026-08-01 | Fase 2.4-2.7 (Governance + vigencias) | Juan Alberto |
| 2026-08-15 | Fase 3.1 (Auditoría bases) | Bibliotecario |
| 2026-09-15 | Fase 3.2-3.3 (SAML + proxy) | Juan Alberto |
| 2026-10-01 | Fase 4.1-4.2 (Metabase + schema) | Juan Alberto |
| 2026-11-01 | Fase 4.3-4.4 (ETL + dashboards) | Juan Alberto + Bibliotecario |
| 2026-12-01 | Go-live producción | Todos |

## Recursos requeridos

### Hardware / Infraestructura
- Servidor SSO (Keycloak): 4 cores, 16GB RAM, 100GB disk
- Servidor Governance (MidPoint): 4 cores, 16GB RAM, 100GB disk
- Servidor Analytics (Metabase): 2 cores, 8GB RAM, 50GB disk
- Proxy bases (Nginx): 2 cores, 4GB RAM, 20GB disk
- IPs públicas: 2-3 (proxy, SSO)

### Personal
- Ing. Juan Alberto Sánchez: 100% dedicado en Fase 1-4 (8-12 meses)
- Bibliotecario UPeU: 20-30% en paralelo (audit, datos matrícula, UAT)
- Admin AD/IT UPeU: 10% puntualmente (validar conectividad, credenciales)

### Licencias / Costos

| Ítem | Costo anual | Notas |
|------|------------|-------|
| MidPoint Community | Gratis | Open source |
| Keycloak | Gratis | Open source, Red Hat patrocinador |
| Metabase Community | Gratis | Open source |
| PostgreSQL | Gratis | Open source |
| Nginx | Gratis | Open source |
| Dominio sso.upeu.edu.pe | $12 | Registrador |
| Certificado SSL | Gratis | Let's Encrypt via Certbot |
| **Total** | **~$12/año** | Hardware en datacenter propio de UPeU |

## Criterios de éxito

### Fase 1
- Usuarios logueados en 4 apps (DSpace, OJS, Koha, Moodle) con una sola contraseña
- 95%+ de reducción en tickets de "password reset"
- Portal funcional, usuarios accesibles

### Fase 2
- Vigencias automáticas: alumnos egresados pierden acceso en 24h de ser marcados en matrícula
- MidPoint sincroniza cambios de facultad/carrera a Keycloak en < 30 min
- 0 usuarios "zombie" (egresados con acceso activo)

### Fase 3
- Acceso remoto a EBSCO, Scopus, ProQuest sin VPN (solo login Keycloak)
- Usuarios de cualquier red (4G, WiFi público, oficina) acceden a bases

### Fase 4
- Dashboard muestra uso real por facultad, carrera, período
- Datos para renovación inteligente de contratos (ej: "Scopus solo usado por Docentes, no por alumnos")
- Identificación de bases underutilized

## Riesgos y mitigación

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|--------|-----------|
| AD de UPeU no accesible desde afuera | Media | Alto | Verificar conectividad previo a Fase 1.2 |
| Sistema de matrícula no tiene API | Media | Alto | Plan B: CSV + cron script en Fase 2.4 |
| CONCYTEC no permite SAML para EBSCO | Baja | Medio | Proxy Nginx comoFallback (Fase 3.3) |
| Usuarios no adoptan SSO | Baja | Medio | Capacitación, comunicación, soporte activo |
| Servidor de governance sale de servicio | Baja | Alto | Backup automático PostgreSQL, replicación standby |

## Próximos pasos inmediatos

1. **ESTA SEMANA**: Validar disponibilidad de Ing. Juan Alberto (100% dedicado Fase 1)
2. **PRÓXIMA SEMANA**: Reunión con Bibliotecario UPeU → confirmar investigación pendiente
3. **SEMANA 3**: Provision de servidores en VMware de UPeU
4. **SEMANA 4**: Inicio Fase 1.1 (Keycloak + PostgreSQL)
