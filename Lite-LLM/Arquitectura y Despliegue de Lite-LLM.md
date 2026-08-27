# Arquitectura y Despliegue de Gateway LLM mediante LiteLLM

## 1. Resumen Ejecutivo

Para responder a los requerimientos de administración y control en el uso de APIs de Inteligencia Artificial, se plantea la implementación de **LiteLLM** como plataforma de gestión centralizada. Esta infraestructura actuará como un Proxy/Gateway unificado para administrar claves de acceso, auditar el consumo de tokens y aplicar políticas de límite de tasa (rate limiting) por clave, optimizando la distribución de recursos y garantizando el control operativo sobre los modelos consumidos.

La integración de LiteLLM resuelve la dispersión en la gestión de credenciales y el descontrol del consumo mediante las siguientes funciones principales:

- **Proxy Unificado e Interfaz OpenAI-Compatible:** Permite estandarizar las peticiones HTTP/REST a través de un único endpoint, adaptando proveedores externos a un formato compatible.
- **Control de Consumo y Quotas:** Monitoreo en tiempo real del uso de tokens y capacidad de definir límites de tasa (rate limits) e importes máximos por API Key emitida.
- **Gobierno de Accesos:** Generación de claves virtuales independientes para usuarios o servicios internos, desacoplándolos de las credenciales maestras del proveedor.

### Integración de Proveedor y Mapeo de Modelos

El despliegue utilizará como proveedor principal la API de **MiniMax**, aprovechando sus capacidades nativas (modelos como MiniMax-M2.7, MiniMax-M3, MiniMax-H3, MiniMax-Speech 2.8 y MiniMax-Music3). A través de LiteLLM, se definirán y expondrán tres aliases de modelos optimizados para casos de uso específicos:

| Nombre del Modelo expuesto | Basado en / Categoría | Propósito y Caso de Uso |
| --- | --- | --- |
| Minimax-m27-highspeed | MiniMax-M2.7 | Modelo principal para generación acelerada de texto y tareas generales de alta velocidad. |
| Minimax-m3 | MiniMax-M3 | Modelo con arquitectura multimodal orientado a tareas de razonamiento avanzado e interpretación de datos (excluyendo generación de imágenes). |
| Minimax-Instatic | Adaptación formato OpenAI | Estructura orientada a compatibilidad estricta con esquemas JSON (JSON Mode / Structured Outputs). |

### Estado Actual de Credenciales y Seguridad

Actualmente, el servicio tiene en operacion bajo una Key provista por MiniMax:

- **Alcance:** Acceso irrestricto a todos los modelos del catálogo MiniMax.
- **Sin Restricciones:** Carece de límites de consumo, cuota mensual o restricción de peticiones por minuto.

**Nota de Seguridad:** Se establece como prioridad dentro del proxy LiteLLM encapsular la API Key maestra de MiniMax en variables de entorno y exponer únicamente claves virtuales (virtual keys) con cuotas limitadas para los consumidores finales.

### Próximos Pasos para la Implementación

- **Aprovisionamiento:** Despliegue del contenedor de LiteLLM junto a una base de datos PostgreSQL en la infraestructura Oracle para guardar métricas y perfiles.
- **Configuración de Modelos (`config.yaml`):** Mapeo explícito de los endpoints de MiniMax y configuración de los tres aliases definidos.
- **Emisión de Claves Virtuales:** Creación de API Keys internas con límites de tasa (rate limiting) asignados por departamento o servicio.

---

## 2. Configuración y Consumo del Gateway LiteLLM mediante Claude Code

Para consumir los modelos expuestos a través del gateway centralizado **LiteLLM**, se establece el uso de la herramienta de línea de comandos **Claude Code** como cliente preferente, sin perjuicio de que pueda utilizarse cualquier otro cliente o gestor compatible con la API de Anthropic/OpenAI. En este documento se detalla el procedimiento de instalación en entornos Windows y la configuración de las variables de entorno necesarias para enrutar las peticiones hacia la infraestructura propia.

### Obtención de Credenciales y Selección de Modelos

Antes de iniciar la configuración local, el usuario debe obtener sus credenciales de acceso y verificar la disponibilidad de los servicios:

- **Generación de API Key:** Ingrese a la consola de administración en [https://llm.seius.com.ec](https://llm.seius.com.ec) para extraer o generar su clave de acceso (Virtual Key).
- **Selección del Modelo:** Verifique en el panel el catálogo de modelos activos y elija el identificador correspondiente según el caso de uso (p. ej., Minimax-m27-highspeed, Minimax-m3 o Minimax-Instatic).

## 3. Procedimiento de Instalación y Configuración

### 3.1 Instalación de Claude Code

Abra una ventana de **Windows PowerShell** (de preferencia con privilegios de administrador) y ejecute el siguiente comando para descargar e instalar el cliente:

```powershell
irm https://claude.ai/install.ps1 | iex
```

### 3.2 Configuración de Variables de Entorno

Para redirigir el tráfico de Claude Code hacia la instancia privada de LiteLLM, es necesario definir el endpoint base, el token de autenticación y el modelo objetivo dentro de la sesión de PowerShell:

```powershell
# Definición de credenciales de acceso
$env:ANTHROPIC_AUTH_TOKEN = "sk-tu-token-aqui"
$env:ANTHROPIC_API_KEY    = "sk-tu-token-aqui"

# Redirección al gateway privado de LiteLLM
$env:ANTHROPIC_BASE_URL   = "https://llm-api.seius.com.ec"

# Identificador del modelo seleccionado
$env:ANTHROPIC_MODEL      = "nombre-del-modelo-elegido"
```

**Nota:** Reemplace `"sk-tu-token-aqui"` por la clave obtenida en el panel llm.seius.com.ec y `"nombre-del-modelo-elegido"` por el alias del modelo deseado.

### Consideraciones Operativas

- **Persistencia de Variables:** Los comandos `$env:` aplicados en PowerShell aplican únicamente para la sesión actual. Para establecer las variables de manera permanente en el sistema operativo, pueden configurarse desde las **Propiedades del Sistema > Variables de Entorno**.
- **Compatibilidad:** Gracias a la capa de abstracción de LiteLLM, las peticiones que realiza Claude Code bajo el formato de Anthropic son traducidas transparentemente hacia la API de MiniMax u otros proveedores respaldados.
