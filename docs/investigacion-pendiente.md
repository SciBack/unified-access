---
title: Preguntas de Investigación Pendientes
author: Ing. Juan Alberto Sánchez
date: 2026-04-07
version: 1.0
status: Investigación
---

# Investigación Pendiente

Preguntas que deben responderse antes de iniciar implementación. Están organizadas por área y asignadas a responsables.

## Infraestructura — TI de UPeU

### Active Directory

**Responsable:** Admin de Sistemas de UPeU

- [ ] **¿Qué versión de AD tienen?**
  - Windows Server 2019, 2022?
  - ¿Alojado on-premises o en Azure (Entra ID)?
  - Respuesta esperada: "Windows Server 2022, on-premises en datacenter Lima"

- [ ] **¿Cuál es el dominio raíz?**
  - Ejemplo: `upeu.edu.pe`
  - Respuesta esperada: "upeu.edu.pe"

- [ ] **¿Cuál es la configuración LDAP?**
  - Server LDAP: `192.168.x.x` o FQDN
  - Puerto: 389 (sin SSL) o 636 (con SSL)?
  - Base DN: `dc=upeu,dc=edu,dc=pe`
  - Respuesta esperada: "ldap://192.168.15.100:389, base DN: dc=upeu,dc=edu,dc=pe"

- [ ] **¿Tenemos cuenta de servicio para LDAP bind?**
  - Usuario: `ldapbind@upeu.edu.pe` o similar
  - ¿Ya existe o hay que crearla?
  - ¿Permisos: solo lectura en usuarios?
  - Respuesta esperada: "Sí, existe ldapbind@upeu.edu.pe con permisos de lectura"

- [ ] **¿Hay Azure AD Connect o hybrid AD?**
  - AD sincroniza a Entra ID
  - ¿Microsoft 365 está activo en UPeU?
  - Respuesta esperada: "Sí, tenemos Microsoft 365, Azure AD Connect sincroniza a Entra ID"

- [ ] **¿Qué atributos de usuario están poblados en AD?**
  - givenName, surname, mail, department
  - ¿Hay campos personalizados para facultad/carrera?
  - Respuesta esperada: "Sí, poblamos: cn, givenName, surname, mail, department; NO tenemos facultad/carrera en AD"

- [ ] **¿La red permite conexión LDAP desde servidores externos?**
  - ¿Hay firewall entre servers Keycloak/MidPoint y AD?
  - ¿Puerto 389/636 abierto?
  - Respuesta esperada: "Sí, firewall permite LDAP desde 192.168.15.0/24"

- [ ] **¿Hay políticas de contraseña vigentes?**
  - Complejidad
  - Expiración
  - ¿Afectaría a SSO?
  - Respuesta esperada: "Complejidad 12 caracteres, expira cada 90 días"

### Bases de datos comerciales

**Responsable:** Jefe de Biblioteca de UPeU

- [ ] **¿Qué bases de datos tienen contratadas?**
  - Nombre: EBSCO, Scopus, ProQuest, Web of Science, etc.
  - Tipo: multidisciplinar, índice de citaciones, especializada
  - Respuesta esperada:
    ```
    - EBSCO Discovery Service (vía CONCYTEC)
    - Scopus (Elsevier, directo)
    - ProQuest Dissertations (directo)
    - Web of Science (Clarivate, directo)
    ```

- [ ] **¿Cuál es el método de autenticación actual de cada base?**
  - IP: rango de IPs de UPeU registradas
  - Credenciales: usuario + password por base
  - SAML: ya implementado
  - Respuesta esperada:
    ```
    - EBSCO: IP (200.xx.xx.xx)
    - Scopus: IP + credenciales de respaldo
    - ProQuest: IP
    - Web of Science: IP
    ```

- [ ] **¿Quién es el contratante de cada base?**
  - CONCYTEC (negociado nacional, sin costo)
  - Directo con proveedor
  - Distribuidor local (ej: Infotec Perú)
  - Respuesta esperada:
    ```
    - EBSCO: CONCYTEC
    - Scopus: EBSCO (distribuidor)
    - ProQuest: Directo
    - Web of Science: CLARIVATE (distribuidor)
    ```

- [ ] **¿Cuál es la vigencia de cada contrato?**
  - ¿Hasta cuándo está vigente?
  - ¿Fecha de renovación?
  - ¿Costo anual (solo si directo)?
  - Respuesta esperada:
    ```
    - EBSCO CONCYTEC: anual, próxima renovación Jan 2027
    - Scopus: anual, vence 31/12/2026, $5,000 USD
    - ProQuest: bienal, vence 30/06/2027, $2,500 USD
    ```

- [ ] **¿El contrato de CONCYTEC permite cambiar a SAML?**
  - ¿Está especificado el método de autenticación?
  - ¿Hay clausúla que lo permita renegociar?
  - Respuesta esperada: "No especifica método, pero CONCYTEC dice que con aviso 60 días se puede cambiar"

- [ ] **¿Hay datos de uso (estadísticas) de cada base?**
  - Número de sesiones mensuales
  - Descargas por período
  - Usuarios únicos
  - Respuesta esperada: "Sí, EBSCO envía reportes mensuales, Scopus tiene portal de admin"

- [ ] **¿Hay soporte técnico designado en UPeU para cada base?**
  - Contacto principal
  - Backup
  - Respuesta esperada: "Lic. Consuelo López, coordinadora de Recursos Digitales"

- [ ] **¿Alguna base tiene WAYFless URL disponible?**
  - Algunos proveedores ofrecen URLs sin redirección adicional
  - Respuesta esperada: "EBSCO no, pero Scopus y ProQuest sí tienen WAYFless"

### Sistema de matrícula

**Responsable:** Director/Coordinador de Informática Académica

- [ ] **¿En qué sistema está la matrícula?**
  - BANNER, ARGOS, Integra, Polyglot, propio
  - Respuesta esperada: "Sistema propio, base de datos SQL Server"

- [ ] **¿Tiene API REST o interfaz de integración?**
  - Documentación disponible
  - Autenticación requerida
  - Respuesta esperada: "Sí, API REST en http://matricula.upeu.edu.pe/api/, autenticación via API key"

- [ ] **¿Qué atributos expone el sistema de matrícula?**
  - Usuario (matrícula)
  - Nombre completo
  - Email
  - Facultad
  - Escuela/Departamento
  - Carrera
  - Nivel (pregrado/posgrado)
  - Estado (activo/egresado/suspendido)
  - Fecha de matrícula
  - Fecha fin de vigencia actual (período)
  - Tipo de usuario (alumno/docente/administrativo)
  - Respuesta esperada:
    ```
    matricula, nombre, email, facultad_id, escuela_id, carrera_id,
    nivel, estado, fecha_matricula, fecha_fin_vigencia, tipo_usuario
    ```

- [ ] **¿Con qué frecuencia se actualiza?**
  - ¿Datos en tiempo real o batch diario?
  - ¿Hay registros históricos?
  - Respuesta esperada: "Actualización en tiempo real, histórico de 5 años"

- [ ] **¿Puede exportar CSV si no hay API?**
  - Formato
  - Frecuencia posible (diaria, semanal)
  - Respuesta esperada: "Sí, puede exportar CSV diariamente a las 2am"

- [ ] **¿El ID de usuario en matrícula coincide con AD?**
  - ¿Es la matrícula el username en AD?
  - ¿Hay campo de email único para correlacionar?
  - Respuesta esperada: "Matrícula diferente a username AD, pero email es único en ambos"

- [ ] **¿Hay cambios frecuentes de carrera/facultad?**
  - ¿Alumnos pueden cambiar de carrera mid-semestre?
  - ¿Con qué frecuencia pasa?
  - Respuesta esperada: "Raro, máximo 3-5 por semestre"

- [ ] **¿Cómo se manejan alumnos en programa doble/triple?**
  - ¿Aparecen en múltiples carreras?
  - Respuesta esperada: "Sí, aparecen en 2 carreras, ambos con mismo estado"

- [ ] **¿Hay docentes, administrativos, otros roles?**
  - ¿Están en el mismo sistema de matrícula?
  - ¿O hay sistema separado de RR.HH.?
  - Respuesta esperada: "Docentes están en matrícula con tipo_usuario='DOCENTE'. Administrativos en sistema RR.HH. separado"

### Infraestructura de red

**Responsable:** Jefe de Infraestructura / Admin Sistemas

- [ ] **¿Dónde se hospedarán los servidores?**
  - Datacenter on-premises
  - AWS / Azure / Google Cloud
  - VPS
  - Respuesta esperada: "Datacenter Lima, VMware ESXi 7.0"

- [ ] **¿Rango de IPs disponibles?**
  - Para Keycloak, MidPoint, Metabase
  - ¿Hay IPs públicas disponibles o detrás de NAT?
  - Respuesta esperada: "Sí, 192.168.15.0/25 disponible, 2 IPs públicas para SSO y Proxy"

- [ ] **¿Hay DNS local o público?**
  - ¿Pueden crear registros para sso.upeu.edu.pe?
  - ¿Qué servidor DNS manejan?
  - Respuesta esperada: "DNS publico (AWS Route53), pueden crear registros nuevos"

- [ ] **¿Conectividad con aplicaciones existentes?**
  - ¿DSpace, OJS, Koha, Moodle están en la misma red?
  - ¿O en servidores separados?
  - Respuesta esperada: "Todos en datacenter Lima, misma red LAN"

- [ ] **¿Hay restricciones de firewall que afecten SAML/OIDC?**
  - HTTPS 443 abierto en ambos sentidos
  - Redireccionamientos HTTP 301
  - Respuesta esperada: "Sí, firewall permite HTTPS. HTTP redirige a HTTPS."

- [ ] **¿Hay backup y disaster recovery policy?**
  - ¿Backups diarios?
  - ¿Replicación?
  - Respuesta esperada: "Sí, backup diario a NAS local, replicación semanal a AWS"

- [ ] **¿Cuál es la disponibilidad esperada (SLA)?**
  - 99.9%, 99.5%, 95%?
  - Ventana de mantenimiento permitida
  - Respuesta esperada: "99.5%, mantenimiento sábado 2am-6am"

## Negocio / Decisión

**Responsable:** Rector o Vicerrector Académico

- [ ] **¿Hay presupuesto asignado para este proyecto?**
  - Hardware, licencias (aunque sean gratuitas)
  - Personal (Ing. Juan Alberto 100%, tiempo del Bibliotecario)
  - Respuesta esperada: "Sí, S/ 50,000 para 2026"

- [ ] **¿Cuál es el timeline esperado?**
  - ¿Go-live en qué fecha?
  - ¿Hay plazos críticos (inicio de semestre, etc.)?
  - Respuesta esperada: "Go-live antes de inicio de semestre 2027-I (agosto 2026)"

- [ ] **¿Este proyecto es estratégico o tactical?**
  - ¿Forma parte de transformación digital mayor?
  - ¿O es para resolver pain point específico?
  - Respuesta esperada: "Estratégico, parte de acreditación digital SUNEDU"

- [ ] **¿Hay actores externos que necesitan coordinación?**
  - SUNEDU, CONCYTEC, otros
  - Respuesta esperada: "Sí, CONCYTEC debe estar al tanto de cambio de auth method"

---

## Tabla de recolección de respuestas

Usar este template para documentar:

```markdown
## [TEMA] — Respondido el [FECHA]

**Pregunta:** ...?
**Respondido por:** Nombre, cargo
**Respuesta:**
> [texto exact o párrafo]

**Decisión / Acción:**
- [ ] Si respuesta es A → hacer X
- [ ] Si respuesta es B → hacer Y

---
```

## Timeline de investigación

| Semana | Tarea | Responsable | Meta |
|--------|-------|-------------|------|
| 1 | Reunión kick-off UPeU | Juan Alberto + Bibliot. | Identificar stakeholders |
| 1-2 | Investigación AD | Admin Sist. UPeU | Enviar respuestas sección AD |
| 1-2 | Investigación matrícula | Dir. Informática Acad. | Enviar respuestas sección Matrícula |
| 2-3 | Investigación bases | Jefe Biblioteca | Enviar respuestas sección Bases |
| 2-3 | Investigación infraestructura | Jefe Infra | Enviar respuestas sección Infra |
| 3 | Análisis de respuestas | Juan Alberto | Documento síntesis |
| 4 | Validación con UPeU | Juan Alberto + Bibliot. | Confirmar supuestos |
| 5 | Aprobación go-ahead | Rectoría | Autorización para Fase 1 |

## Documento a generar después

Una vez respondidas todas las preguntas:

**`docs/especificacion-tecnica-upeu.md`**

Documento que sintetiza:
- Arquitectura específica para UPeU (no genérica)
- Integraciones confirmadas
- IPs, DNS, dominios reales
- Consideraciones especiales por CONCYTEC
- Cronograma ajustado a UPeU
- Presupuesto final
- Aprobación de stakeholders
