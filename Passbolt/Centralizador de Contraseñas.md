**Implementación de Gestor Centralizado de Contraseñas (Passbolt)**

La gestión ineficiente y la compartición no segura de credenciales
representan un riesgo crítico para la confidencialidad de la
información, incrementando la exposición a filtraciones. Para mitigar
esta vulnerabilidad, resulta indispensable implementar una solución
centralizada de gestión de contraseñas con control de acceso basado en
roles (RBAC). Este esquema permite asignar o revocar accesos a grupos de
recursos según las funciones de cada usuario, optimizando el ciclo de
vida de los accesos corporativos.

**Selección de la Solución**

Se determinó que **Passbolt** es la alternativa óptima para resolver
estas falencias, basándose en los siguientes factores:

-   **Despliegue Local (Self-Hosted):** Capacidad de alojamiento e
    integración directa dentro de la infraestructura VPS en Oracle Cloud
    Infrastructure.

-   **Gestión por Grupos y Permisos:** Administración granular de
    usuarios y asignación de credenciales según roles.

-   **Facilidad de Uso:** Interfaz administrativa intuitiva que minimiza
    la curva de aprendizaje y reduce errores operacionales.

**Configuración y Modelo de Seguridad**

-   **Servicio de Notificación (SMTP):** Se integró el servicio de
    correo transaccional de **Resend** para el envío automatizado de
    invitaciones y credenciales de acceso a los usuarios.

-   **Responsabilidad de la Clave Privada/Maestra:** Cada usuario genera
    una clave única e intransferible durante su registro. Bajo el
    esquema de cifrado de Passbolt, **la pérdida de esta clave implica
    la pérdida total e irrecuperable de la cuenta**, recayendo la
    custodia de este factor exclusivamente en el usuario final.

**Estrategia de Respaldos e Integridad de Datos**

Para garantizar la continuidad del negocio y la recuperación ante
desastres (DRP) en caso de un fallo crítico en el VPS, se definió una
arquitectura de respaldo dedicada:

-   **Frecuencia:** Copias de seguridad mensuales programadas de la base
    de datos de Passbolt.

-   **Cifrado y Almacenamiento:** Los respaldos se cifran localmente
    antes de ser transferidos de forma segura a un *bucket* de
    almacenamiento de objetos en **Cloudflare R2**.
