---
title: Arquitectura de Unified Access — UPeU
author: Ing. Juan Alberto Sánchez
date: 2026-04-07
version: 1.0
status: Investigación
---

# Arquitectura — Unified Access UPeU

Descripción detallada de cada capa del sistema de acceso unificado.

## Visión general

El sistema se organiza en **6 capas verticales**:

1. **Capa de identidad**: Fuentes de verdad (AD, Entra ID, matrícula)
2. **Capa de governance**: MidPoint (sincronización, roles, vigencias)
3. **Capa de autenticación**: Keycloak (SSO - SAML/OIDC)
4. **Capa de aplicaciones**: DSpace, OJS, Koha, Moodle (clientes SAML/OIDC)
5. **Capa de acceso remoto**: Nginx proxy para bases externas (IP auth + Keycloak)
6. **Capa de analítica**: Colector de eventos → PostgreSQL → Metabase

## Capa 1: Identidad — Fuentes de verdad

La fuente de verdad es la autoridad sobre quién es cada usuario y qué atributos tiene.

### Estructura de identidades

**Fuentes primarias:**

1. **Active Directory on-premises** (UPeU ya tiene)
   - Identidades base: usuario, email, nombre, apellidos
   - Estado activo/inactivo
   - Pertenencia a grupos de seguridad (pueden mapearse a roles)
   - Almacenamiento centralizado de contraseñas (Kerberos)

2. **Entra ID / Azure AD** (opcional si UPeU tiene Microsoft 365)
   - Replica identidades desde AD on-premises (Azure AD Connect)
   - Proporciona MFA (multi-factor authentication)
   - Sincronización bidirecional

3. **Sistema de matrícula** (CSV o API)
   - Roles académicos por período: alumno, docente, administrativo
   - Atributos por usuario:
     - Facultad (ej: "Facultad de Ingeniería")
     - Escuela (ej: "Ingeniería de Sistemas")
     - Carrera (ej: "Ingeniería de Sistemas Informáticos")
     - Nivel (ej: "Pregrado", "Posgrado")
     - Estado (ej: "Activo", "Egresado", "Suspendido")
     - Fecha de matrícula
     - Fecha fin de vigencia (período académico actual)

### Mapeo de atributos

```
Active Directory           Sistema de matrícula    Keycloak (atributos SAML)
└─ uid (usuario)          └─ matricula_id         └─ uid (identidad única)
└─ mail                      carrera               └─ email
└─ givenName               facultad               └─ givenName
└─ surname                 escuela                └─ surname
└─ company                 nivel                  └─ company
└─ departmentNumber        estado                 └─ department
                           fecha_fin_vigencia     └─ fecha_fin_vigencia
```

### Flujo de sincronización

```
AD on-premises
    ↓ (LDAP query cada 30 min)
MidPoint
    ↓ (correlaciona con matrícula)
    ├─ Enriquece con facultad, carrera, vigencia
    ├─ Calcula roles derivados (alumno_2024_II, docente_activo, etc.)
    └─ Provisiona a Keycloak (vía API)
    ↓
Keycloak
    ↓ (mapeo en SAML assertions)
Aplicaciones (DSpace, OJS, Koha, Moodle)
```

## Capa 2: Governance — MidPoint

MidPoint es un Identity Governance and Administration (IGA) que sincroniza identidades, gestiona roles y valida vigencias.

### Responsabilidades de MidPoint

1. **Sincronización de fuentes**
   - Conectar a AD (LDAP)
   - Consultar sistema de matrícula (CSV, API REST, ODBC)
   - Fusionar datos: AD como identidad principal + matrícula como atributos contextuales

2. **Gestión de roles**
   - Rol automático: `alumno` si en matrícula aparece como alumno activo
   - Rol automático: `docente` si matrícula lo marca como docente en período vigente
   - Rol automático: `administrativo` si pertenece a ciertos grupos de AD
   - Rol automático: `egresado` si fecha_fin_vigencia < hoy
   - Rol derivado: `puede_acceder_a_EBSCO` si es alumno O docente (no egresado)
   - Rol derivado: `puede_acceder_a_bases_limitadas` si es alumno de pregrado

3. **Vigencias automáticas**
   - Acceso activo: mientras alumno está matriculado en período vigente
   - Acceso suspendido: cuando egresa (fecha_fin_vigencia < hoy)
   - Deprovisioning automático: cambios sin intervención manual

4. **Provisioning/Deprovisioning**
   - Crear usuario en Keycloak cuando ingresa
   - Actualizar atributos en Keycloak cuando cambia de carrera/facultad
   - Deshabilitar usuario en Keycloak cuando egresa
   - Sincronizar grupos y roles derivados

5. **Portal de autogestión** (para bibliotecarios)
   - Búsqueda de usuario
   - Vista de atributos actuales
   - Histórico de cambios de vigencia
   - Forzar sincronización manual
   - Ver estado de provisioning en cada aplicación

6. **Auditoría**
   - Log de todos los cambios de identidad
   - Quién cambió qué, cuándo, por qué
   - Rastro de deprovisioning

### Arquitectura de MidPoint

```
MidPoint (Docker)
├─ Connector LDAP → AD on-premises
├─ Connector CSV → matrícula (polling automático)
├─ Connector OIDC → Keycloak (vía API)
├─ Expresiones de roles (business logic)
├─ Políticas de vigencia
└─ Portal de autogestión
    └─ PostgreSQL (audit log, cache)
```

### Ejemplo: Flujo de alumno egresado

1. MidPoint consulta matrícula diariamente
2. Detecta que fecha_fin_vigencia de usuario X = 2024-12-15 (pasó)
3. Calcula: rol `alumno` → NO aplica, rol `egresado` → SÍ aplica
4. Actualiza Keycloak: deshabilita roles de acceso a bases de pago
5. Log de auditoría: "Usuario X deprovisioned — egresado desde 2024-12-15"
6. Próximo login del usuario: se autentica pero no ve EBSCO, Scopus, etc.

## Capa 3: Autenticación — Keycloak

Keycloak es el Identity Provider (IdP) central. Todos los sistemas confían en él para autenticar usuarios.

### Federación de identidades

Keycloak se conecta con:

1. **LDAP (Active Directory)**
   - Valida credenciales contra AD
   - Usuario escribe "usuario@upeu" y contraseña
   - Keycloak consulta LDAP para verificar
   - También consume atributos de AD (nombre, email)

2. **OIDC (Entra ID)** — opcional si UPeU tiene Azure AD
   - Keycloak actúa como cliente OIDC de Entra ID
   - Usuario se autentica con MFA en Entra ID
   - Entra ID redirige a Keycloak con token
   - Keycloak acepta la autenticación

### Flujo SAML de Keycloak

```
Usuario en navegador
       ↓ (accede a DSpace)
DSpace (SP = Service Provider)
       ↓ (redirect a Keycloak con AuthnRequest SAML)
Keycloak (IdP = Identity Provider)
       ↓ (login form o redirige a AD/Entra ID)
AD / Entra ID
       ↓ (credenciales validadas)
Keycloak
       ↓ (genera Response SAML firmado con atributos)
DSpace (valida firma, extrae uid + email + roles)
       ↓ (crea sesión)
Usuario autenticado en DSpace
```

### Mapeo de atributos SAML

Keycloak mapea atributos de MidPoint a SAML assertions:

```
SAML Attribute Name        Valor                  Consumidor
─────────────────────────────────────────────────────────
urn:oid:0.9.2342.19200300.100.1.3  usuario@upeu     [todos]
urn:oid:0.9.2342.19200300.100.1.1  juan.sanchez     [todos]
urn:oid:2.5.4.3            Juan Sanchez           [todos]
mail                       juan@upeu.edu.pe       [todos]
urn:oid:2.5.4.97           Facultad de Ing.       [DSpace analytics]
groups                     alumno_2024_II,        [app-specific]
                           puede_acceder_EBSCO    
fecha_fin_vigencia         2025-12-15             [MidPoint decision]
```

### Clientes SAML configurados en Keycloak

| Aplicación | Tipo | Protocol | Descripción |
|------------|------|----------|-------------|
| DSpace | Interno | SAML 2.0 | Repositorio institucional |
| OJS | Interno | OIDC | Revista institucional |
| Koha | Interno | SAML 2.0 | Sistema de biblioteca |
| Moodle | Interno | SAML 2.0 | LMS |
| EBSCO | Externo | SAML 2.0 | Base de datos académica |
| Scopus | Externo | SAML 2.0 | Índice de citaciones |
| ProQuest | Externo | SAML 2.0 | Bases académicas |
| Web of Science | Externo | SAML 2.0 | Índice de citaciones |

## Capa 4: Aplicaciones internas

Cada aplicación integrada con Keycloak consume assertions SAML/OIDC.

### DSpace 7.6.6

- Método: SAML 2.0
- Configuración: `dspace/config/modules/authentication-saml.cfg`
- Atributos mapeados:
  - `urn:oid:0.9.2342.19200300.100.1.3` → uid
  - `mail` → email
  - `groups` → comunidad/colección (permisos de curador)
  - Facultad → metadato `dc.contributor.corporateName`

### OJS 3.x

- Método: OIDC (plugin de PKP)
- Configuración: `config.inc.php` sección `[oidc]`
- Atributos mapeados:
  - `sub` → username
  - `email` → email
  - `name` → nombre
  - Rol: mapeo automático (autor, editor, revisor basado en grupo en Keycloak)

### Koha

- Método: SAML via módulo de terceros
- Configuración: `etc/koha-conf.xml`
- Atributos: uid, email, nombre
- SSO transparente para estudiantes y docentes

### Moodle

- Método: SAML (módulo nativo)
- Configuración: `auth/saml/settings.php`
- Atributos: uid, email, nombre, rol (student/teacher basado en grupo)
- Sincronización de cursos: via API de Moodle enriquecida con datos de matrícula

## Capa 5: Acceso remoto — Proxy Nginx

Para bases de datos externas que **no soportan SAML** o que funcionan solo con IP institucional.

### Método 1: SAML WAYFless (bases con SAML nativo)

Para EBSCO, Scopus, ProQuest que soportan SAML:

1. Usuario accede a WAYFless URL desde fuera del campus:
   ```
   https://bases.upeu.edu.pe/ebsco/?target=https://search.ebscohost.com/...
   ```

2. Nginx redirige a Keycloak:
   ```
   https://sso.upeu.edu.pe/auth/realms/upeu/protocol/saml/clients/ebsco
   ```

3. Usuario se autentica (LDAP via Keycloak)

4. SAML Response redirige a EBSCO con atributos y IP institucional

5. EBSCO reconoce el SAML assertion y permite acceso

**Beneficio**: usuario accede desde cualquier red sin VPN

### Método 2: IP proxy (bases solo-IP)

Para bases que **solo aceptan autenticación por IP**:

1. Usuario accede a URL en portal UPeU:
   ```
   https://portal.upeu.edu.pe/recursos/base-xy
   ```

2. Portal verifica autenticación en Keycloak

3. Portal redirige a Nginx proxy (que tiene IP institucional registrada en la base)

4. Nginx sale hacia la base con su IP pública (institucional)

5. Base recibe petición desde IP UPeU → acceso permitido

**Configuración Nginx:**

```nginx
server {
    listen 443 ssl;
    server_name proxy.upeu.edu.pe;

    # verificar autenticación Keycloak
    location / {
        auth_request /auth;
        auth_request_set $auth_user $upstream_http_x_user;
        
        # proxy a la base (con IP institucional de Nginx)
        proxy_pass http://<backend>;
        proxy_set_header X-Forwarded-For $auth_user;
    }

    location /auth {
        # mini-server que valida contra Keycloak
        proxy_pass http://keycloak:8080/auth/token;
    }
}
```

## Capa 6: Analítica

Recolección de eventos y visualización de uso.

### Fuentes de datos

1. **Keycloak events**
   - Login exitoso / fallido
   - Logout
   - Cambio de contraseña
   - Timestamp, usuario, IP cliente

2. **Nginx logs** (acceso remoto)
   ```
   $remote_addr - $remote_user [$time_local] 
   "$request" $status $body_bytes_sent "$http_referer"
   ```
   - URL accedida
   - Código HTTP
   - Bytes transferidos
   - Duración de sesión

3. **DSpace statistics**
   - Vistas de ítems
   - Descargas por ítem
   - Usuario, timestamp, IP

4. **OJS stats**
   - Vistas de artículos
   - Descargas de PDF
   - Por usuario, por artículo

### ETL — Extracción, Transformación, Carga

Script Python que corre cada noche:

```python
# extrae.py — Logstash o script Python
1. Consulta Keycloak API por eventos del día
2. Lee logs de Nginx
3. Consulta API de DSpace por descargas
4. Consulta API de OJS por vistas
5. Normaliza: usuario + timestamp + recurso + acción
6. Carga en PostgreSQL tabla `analytics.eventos`
```

### Schema de analytics

```sql
CREATE TABLE analytics.eventos (
    id SERIAL PRIMARY KEY,
    fecha DATE,
    usuario VARCHAR(255),
    facultad VARCHAR(255),
    carrera VARCHAR(255),
    nivel VARCHAR(50), -- pregrado/posgrado
    tipo_evento VARCHAR(50), -- login, descarga, vista, logout
    recurso VARCHAR(255), -- DSpace, EBSCO, Scopus, etc.
    item_id VARCHAR(255), -- uuid de ítem descargado
    accion VARCHAR(255), -- download, view
    timestamp TIMESTAMP,
    ip_cliente INET,
    duracion_sesion_minutos INT
);

CREATE INDEX idx_eventos_fecha ON analytics.eventos(fecha);
CREATE INDEX idx_eventos_usuario ON analytics.eventos(usuario);
CREATE INDEX idx_eventos_facultad ON analytics.eventos(facultad);
CREATE INDEX idx_eventos_recurso ON analytics.eventos(recurso);
```

### Dashboard Metabase

Dashboards públicos para bibliotecarios:

1. **Resumen por mes**
   - Logins totales
   - Descargas totales
   - Bases de datos más usadas
   - Facultades con mayor uso

2. **Por facultad**
   - Uso de cada base por facultad
   - Top 10 documentos descargados
   - Cohorte por nivel (pregrado vs posgrado)

3. **Por carrera**
   - Comparativa de uso entre carreras
   - Recursos especializados

4. **Por recurso**
   - Licencias vs uso real
   - Alertas si uso es anómalo (picos)

## Flujo end-to-end: Alumno accede a EBSCO

1. **Alumno abre navegador fuera del campus** (red móvil, casa)
2. **Accede al portal UPeU**: `portal.upeu.edu.pe`
3. **Portal redirige a Keycloak login page**
4. **Alumno ingresa usuario y contraseña**
5. **Keycloak valida contra LDAP (AD)**
6. **AD confirma credenciales válidas**
7. **MidPoint verifica en matrícula: alumno activo en período 2024-II**
8. **Keycloak genera SAML Response con atributos:**
   ```xml
   <Assertion>
     <Subject>
       <NameID>alumno123</NameID>
     </Subject>
     <AttributeStatement>
       <Attribute Name="mail">alumno@upeu.edu.pe</Attribute>
       <Attribute Name="groups">alumno_2024_II</Attribute>
       <Attribute Name="fecha_fin_vigencia">2024-12-15</Attribute>
     </AttributeStatement>
   </Assertion>
   ```
9. **Keycloak redirige al portal** con SAML assertion
10. **Portal extrae atributos, crea sesión del usuario**
11. **Usuario hace clic en "EBSCO"** en el portal
12. **Portal redirige a WAYFless URL**: 
    ```
    https://bases.upeu.edu.pe/ebsco/?target=https://search.ebscohost.com...
    ```
13. **Nginx proxy redirige a Keycloak** (client=ebsco)
14. **Keycloak reutiliza sesión existente** (ya está logueado)
15. **Keycloak envía SAML Response a EBSCO** (vía navegador)
16. **EBSCO valida SAML**, reconoce alumno de UPeU
17. **EBSCO redirige a búsqueda**, alumno logueado
18. **Alumno busca y descarga artículo**
19. **EBSCO registra descarga** (evento)
20. **Esa noche, ETL recolecta eventos** de Keycloak + EBSCO + Nginx
21. **Metabase muestra**: "Alumno Juan Sanchez (Carrera Ing. Sistemas) descargó 5 artículos de EBSCO en abril"

## Decisiones de diseño

### ¿Por qué MidPoint?

- MidPoint Community Edition es gratuita (y mantenida por Evolveum)
- Soporta governance avanzada: roles derivados, vigencias, auditoría
- Conectores para múltiples fuentes (AD, LDAP, OIDC, SOAP, REST, ODBC)
- Portal de autogestión integrado
- Mejor que sincronizar manualmente usuarios entre sistemas

### ¿Por qué Keycloak?

- Open source (patrocinado por Red Hat)
- Soporte nativo SAML 2.0 y OIDC
- Federación con LDAP, OIDC, SAML upstream
- Escalable (puede atender 10,000+ usuarios)
- Portal de autogestión de perfil
- Fácil de configurar (JSON)

### ¿Docker Compose, no Kubernetes?

- En esta etapa, UPeU no necesita Kubernetes
- Docker Compose es suficiente para 2,000-5,000 usuarios
- Simplifica operaciones (no requiere DevOps especializado)
- Si crece, escalar a Kubernetes es directo

### ¿PostgreSQL en Docker?

- Para fases 1-3, PostgreSQL en Docker funciona bien
- Backups simples (pg_dump)
- Si crece a 10+ clientes (SciBack), evaluar RDS AWS

### ¿Por qué no ssocircle o Shibboleth?

- Shibboleth es más complejo de mantener (Java, compilación)
- ssocircle es cloud (no control local)
- Keycloak + MidPoint es el estándar moderno en educación
- Universidades españolas y latinoamericanas ya usan esta pila

## Roadmap de implementación

Ver [roadmap.md](roadmap.md) para el plan fase por fase.
