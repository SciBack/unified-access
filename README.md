# unified-access-upeu — Acceso Unificado a Recursos de Biblioteca Digital

Alternativa open source a EZproxy para la Universidad Peruana Unión (UPeU). Proporciona acceso unificado mediante SSO a todos los recursos digitales institucionales: DSpace, OJS, Koha, Moodle y bases de datos externas (EBSCO, Scopus, ProQuest, Web of Science).

## Qué es

**unified-access-upeu** es un sistema de acceso unificado que centraliza la autenticación e identidad de usuarios en la Universidad Peruana Unión. Mediante SSO (Single Sign-On), los estudiantes, docentes y administrativos acceden a todos los recursos digitales con una única cuenta, eliminando la necesidad de múltiples credenciales.

Características principales:

- **Autenticación centralizada**: Keycloak como IdP (Identity Provider)
- **Control de acceso granular**: Por vigencia, rol, facultad, carrera y nivel académico
- **Governance de identidades**: MidPoint gestiona roles, provisioning y vigencias automáticas
- **Integración con recursos internos**: DSpace 7, OJS, Koha, Moodle (SAML/OIDC)
- **Acceso a bases de datos comerciales**: EBSCO, Scopus, ProQuest, Web of Science (SAML WAYFless o proxy)
- **Analítica de uso**: Dashboard con dimensiones por facultad, carrera, recurso y período
- **Auditoría completa**: Logs de acceso, intentos fallidos, cambios de rol

## Problema que resuelve

**Estado actual en UPeU:**
- Los usuarios deben loguearse en cada plataforma por separado (DSpace, OJS, Koha, Moodle)
- No hay control centralizado de quién accede a recursos de pago
- No hay vigencia automática de acceso (cuando un alumno egresa, sigue teniendo acceso)
- No hay visibilidad de uso de licencias vs disponibles
- Los reportes de descarga no se correlacionan con datos institucionales (facultad, carrera, nivel)
- Acceso remoto a bases externas requiere mecanismos manuales o anticuados

**Soluciones proporcionadas:**
- Un único login para todos los sistemas
- Vigencia automática: acceso activo mientras el usuario está matriculado
- Deprovisioning automático al egresar o cambiar de rol
- Visibilidad de uso de cada base de datos por facultad y carrera
- Acceso remoto seguro mediante proxy para bases solo-IP
- Reducción de tickets de soporte por contraseñas olvidadas

## Arquitectura general

```
Fuentes de verdad
  ├─ Active Directory on-premises (identidades base)
  ├─ Entra ID / Azure AD (si aplica)
  └─ Sistema de matrícula (roles académicos, vigencias)
            ↓
        MidPoint
   (governance, provisioning, 
    vigencias, roles, auditoría)
            ↓
        Keycloak
   (SSO — SAML 2.0 / OIDC)
            ↓
Aplicaciones internas
  ├─ DSpace 7.6.6 (repositorio)
  ├─ OJS 3.x (revista)
  ├─ Koha (biblioteca)
  ├─ Moodle (LMS)
  └─ Portal de recursos
            ↓
Bases de datos externas (comerciales)
  ├─ EBSCO (SAML WAYFless o IP)
  ├─ Scopus (SAML WAYFless o IP)
  ├─ ProQuest (SAML WAYFless o IP)
  └─ Web of Science (SAML WAYFless o IP)
            ↓
Colector de eventos
  ├─ Keycloak events (login, logout, fallos)
  ├─ Nginx logs (acceso remoto)
  ├─ DSpace statistics (descargas, vistas)
  └─ OJS stats
            ↓
Metabase / Grafana
(dashboard analítico por facultad, 
 carrera, recurso, período)
```

## Stack tecnológico

- **Identidad**: Active Directory + MidPoint + Keycloak
- **Orquestación**: Docker Compose (Ubuntu 22.04 LTS)
- **Datos**: PostgreSQL
- **Proxy**: Nginx
- **Analítica**: Metabase
- **Control de versiones**: Git (GitHub)

## Estado actual

Proyecto en fase de **diseño e investigación**. No hay código en producción.

Próximos pasos:
1. Levantar ambientes de prueba (Keycloak, MidPoint) en servidor de desarrollo
2. Recopilar información de UPeU sobre infraestructura actual (AD, contrataciones de bases)
3. Documentar procesos de integración con cada plataforma
4. Implementar Fase 1 (fundación — Keycloak + DSpace + OJS + Koha + Moodle)

## Documentación

- **[docs/arquitectura.md](docs/arquitectura.md)** — Descripción detallada de cada capa del sistema
- **[docs/bases-externas.md](docs/bases-externas.md)** — Integración con proveedores comerciales (EBSCO, Scopus, ProQuest, Web of Science)
- **[docs/roadmap.md](docs/roadmap.md)** — Plan de implementación en 4 fases
- **[docs/investigacion-pendiente.md](docs/investigacion-pendiente.md)** — Preguntas que deben responderse en UPeU

## Licencia

Este proyecto utiliza componentes open source con licencias respectivas:
- **MidPoint**: GNU General Public License v3
- **Keycloak**: Apache License 2.0
- **Metabase**: GNU Affero General Public License v3
- **Koha**: GNU General Public License v3

Véase cada dependencia para detalles completos.

## Contacto

Proyecto desarrollado para la Universidad Peruana Unión (UPeU).
Infraestructura TI — Ing. Juan Alberto Sánchez
