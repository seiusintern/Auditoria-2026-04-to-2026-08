# Odoo — Documento Consolidado

> Unión de:
> 1. **Arquitectura de Odoo.docx** — documento de arquitectura e implementación.
> 2. **informe_actividad_odoo_2026-08-27.pdf** — informe de actividad al 27‑ago‑2026.
>
> Generado el 2026‑08‑27.

---

## Parte 1 — Arquitectura e Implementación de Odoo Enterprise Espejo para la Gestión Descentralizada de Proyectos y Recursos

Para superar las restricciones de licencias por usuario del CRM principal (Odoo SaaS) y maximizar la eficiencia operativa de Seius, se diseñó e implementó una arquitectura en espejo basada en **Odoo Enterprise** autohospedado en un servidor VPS de Oracle. Esta instancia secundaria descentraliza los flujos de gestión de proyectos, seguimiento de tareas y registro de partes de horas (*timesheets*), incorporando lógica personalizada para el cálculo de avances según el consumo de recursos estimado.

### Antecedentes y Justificación Técnica

#### Problemática de Licenciamiento y Escalabilidad

El sistema Odoo SaaS principal opera como el repositorio central de activos, procesos clave y datos maestros de la organización. Sin embargo, el esquema de suscripción por usuario activo limita el escalamiento operativo para personal técnico, de soporte y de campo.

#### Estrategia de Descentralización (Instancia Espejo)

Para mitigar el costo recurrente de licencias sin comprometer la operativa, se desplegó un entorno autónomo en la infraestructura de Oracle dedicado a:

- **Operación de Proyectos:** Control de tareas, asignación de actividades y registro de partes de horas.
- **Autonomía Operativa:** Desconexión del consumo de licencias del SaaS principal para perfiles técnicos y operativos.
- **Personalización del Modelo de Datos:** Flexibilidad para crear campos customizados sin alterar la base de datos principal.

### Arquitectura del Sistema y Personalización del Modelo

#### Módulo Personalizado: Estimación de Recursos y Avance Neto

En el modelo de tareas (`project.task`) de la instancia espejo se incorporó un nuevo atributo técnico para cuantificar el costo o alcance estimado:

- **Variable de Estimación de Recursos:** Campo customizado que soporta métricas heterogéneas (horas, metros, peso en kg, etc.).
- **Cálculo de Avance Neto:** El registro periódico de horas y avance físico se valida contra esta estimación base, generando un porcentaje real de progreso ponderado por recurso consumido.

#### Gestión de Usuarios e Identidades

Se realizó la sincronización y migración de datos de personal desde la instancia principal hacia la base de datos espejo, resultando en un total de **20 usuarios registrados**, de los cuales **14 se encuentran activos** (`t=true`):

| ID  | Login / Correo Electrónico | Rol / Notas |
|----:|----------------------------|-------------|
| 2   | tics@seius.com.ec          | Administrador Técnico |
| 6   | info@seius.com.ec          | Cuenta General |
| 8   | kevinpaillacho251@gmail.com| Usuario Técnico |
| 16  | eurbina@seius.com.ec       | Usuario Operativo |
| 17  | control@seius.com.ec       | Control Interno |
| 19  | tecnico2@seius.com.ec      | Soporte Técnico |
| 21  | automatizacion@seius.com.ec| Integraciones / Bots |
| 22  | seius.tecnico5@gmail.com   | Técnico de Campo |
| 23  | seiusproyectos@gmail.com   | Gestión de Proyectos |
| 24  | power@seius.com.ec         | Analítica / BI |
| 25  | seius.tecnico4@gmail.com   | Técnico de Campo |
| 26  | lurbina@seius.com.ec       | Usuario Operativo |
| 27  | proyectoseius@gmail.com    | Proyectos |
| 30  | soporteseius@gmail.com     | Mesa de Ayuda |
| 31  | seius.tecnico6@gmail.com   | Técnico de Campo |

### Infraestructura, Seguridad y Servicios Integrados

#### Publicación Web y Cifrado SSL

- **FQDN / URL de Acceso:** [https://odoo.seius.com.ec](https://odoo.seius.com.ec) (Estado: `200 OK`).
- **Proxy Inverso:** Traefik gestiona el enrutamiento HTTP/HTTPS y la terminación de certificados.
- **Seguridad Transport Layer:** Certificado TLS emitido por *Google Trust Services WE1* (vía Let's Encrypt), con vigencia desde el **15 de julio de 2026** hasta el **13 de octubre de 2026** y ciclo de renovación [automática]{.underline} mediante desafío ACME en Traefik.

#### Servicio de Mensajería Saliente (SMTP)

La instancia cuenta con la integración de un servidor SMTP dedicado para:

- Envío de notificaciones automáticas sobre asignación y cambios de estado en tareas.
- Emisión de boletines y retroalimentación (*feedback*) sobre el estado del servicio hacia el personal interno.

---

## Parte 2 — Informe de Actividad de Odoo (2026‑08‑27)

> **Nota sobre el origen:** el PDF adjunto es esencialmente un **dashboard gráfico** (tarjetas KPI, mapas de calor, mini‑gráficos de tendencia). El texto extraíble es limitado; lo que sigue reproduce fielmente las etiquetas, valores y elementos textuales que el PDF expone. Las zonas puramente gráficas se describen como tales en lugar de inventarse contenido.

### Resumen de la instancia (página 1)

| Atributo | Valor |
|---|---|
| Identificador de instancia | `instance-20260506-1423` |
| Base de datos | `seius` |
| Proyecto (`project_project`) | activo |
| `privacy_visibility` | `portal` |
| Estado | 🟢 operativo |

### Tareas (`project_task`) — página 2

| Estado | Cuenta |
|---|---|
| `01_in_progress` | 1 |
| `1_done` | 1 |
| `03_approved` | 1 |

> ⚠ Indicador de atención presente en la tarjeta.

### Partes de horas (`account_analytic_line`) — página 3 y 4

Usuarios con registros visibles en la tarjeta:

- `tics@seius.com.ec`
- `seius.tecnico5@gmail.com`
- `control@seius.com.ec`
- `info@seius.com.ec`
- `power@seius.com.ec`
- `kevinpaillacho251@gmail.com`

Datos complementarios extraídos:

- `tics@`
- `invoice`
- `unit_amount=1 invoice`
- `invoice`
- `other`
- `active=true`

### Usuarios — actividad reciente (página 5)

El panel contrasta usuarios con y sin actividad reciente. Los marcados con ❌ no registran movimiento en el periodo mostrado:

**Listado principal**

| Correo | Actividad reciente |
|---|---|
| tics@seius.com.ec | ✅ |
| power@seius.com.ec | ✅ |
| control@seius.com.ec | ✅ |
| kevinpaillacho251@gmail.com | ❌ |
| eurbina@seius.com.ec | ❌ |
| tecnico2@seius.com.ec | ❌ |
| automatizacion@seius.com.ec | ❌ |
| seius.tecnico5@gmail.com | ❌ |
| seiusproyectos@gmail.com | ❌ |
| lurbina@seius.com.ec | ❌ |
| proyectoseius@gmail.com | ❌ |
| soporteseius@gmail.com | ❌ |
| seius.tecnico6@gmail.com | ❌ |
| seius.tecnico4@gmail.com | ❌ |

**Top 3 con actividad (panel inferior):** `tics@seius.com.ec`, `power@seius.com.ec`, `control@seius.com.ec`.

### Mapa de calor de actividad diaria (página 6)

Frecuencia de eventos por día, del 14 al 27 de agosto de 2026:

| Fecha | Eventos | | Fecha | Eventos |
|---|---:|---|---|---:|
| 2026-08-14 | 1 | | 2026-08-21 | 1 |
| 2026-08-15 | 1 | | 2026-08-22 | 0 |
| 2026-08-16 | 1 | | 2026-08-23 | 0 |
| 2026-08-17 | 2 | | 2026-08-24 | 1 |
| 2026-08-18 | 1 | | 2026-08-25 | **3 ← pico** |
| 2026-08-19 | 2 | | 2026-08-26 | 2 |
| 2026-08-20 | 1 | | 2026-08-27 | **2 ← hoy** |

> *Ago‑2026 = sólo 27 días transcurridos.*

**Rangos de IP de acceso observados:**

- `104.23.x`
- `172.68-71.x`
- `198.41.x`
- `10.0.1.x`

**Indicadores de servicio (página 6):** 6 verdes 🟢 + 1 amarillo 🟡 — tendencia general al alza (📈).

### Tendencia mensual de carga (página 7)

- Pico coincidente con evento de *boot* del servicio (`↑ pico (boot)`).
- Sparkline: `▁▂▄▆████▇▆▄▃▂▁` recorriendo `jun ──────► jul ──────► ago`.
- ⚠ 2 indicadores amarillos 🟡 pendientes de revisión.

### Cabecera del último panel (página 8)

- `admin`

---

## Resumen ejecutivo

- **Arquitectura:** instancia espejo de Odoo Enterprise en VPS Oracle, detrás de Traefik con TLS válido (Google Trust Services WE1) hasta el 13‑oct‑2026.
- **Modelo de datos:** campo custom de *estimación de recursos* en `project.task` con cálculo de *avance neto* ponderado.
- **Identidades:** 20 usuarios migrados, 14 activos en la instancia.
- **Operación al 27‑ago‑2026:** 3 tareas (1 in_progress, 1 done, 1 approved); actividad reciente concentrada en 3 cuentas (`tics`, `power`, `control`); pico de eventos el 25‑ago (3); 1 alerta amarilla y 2 adicionales en el panel de tendencia.
