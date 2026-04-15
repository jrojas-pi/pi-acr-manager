# PI ACR Manager

Flujo de trabajo end-to-end para construir imágenes Docker, subirlas a Azure Container Registry (ACR) y actualizar Azure Container Apps.

## Descripción

PI ACR Manager es un skill diseñado para automatizar el ciclo completo de despliegue de contenedores en Azure:

1. **Build** — Construye la imagen Docker de forma local o en la nube (ACR Tasks).
2. **Push** — Sube la imagen al registro de contenedores de Azure.
3. **Update** — Actualiza la Container App con la nueva imagen.
4. **Verify** — Verifica que el despliegue fue exitoso.

## Prerrequisitos

- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) instalado y autenticado (`az login`).
- Una suscripción de Azure con los siguientes recursos aprovisionados:
  - Azure Container Registry (ACR)
  - Azure Container App
- (Opcional) [Docker](https://docs.docker.com/get-docker/) instalado localmente. Si Docker no está disponible, el skill utiliza ACR Tasks para construir la imagen en la nube.
- Un `Dockerfile` en la raíz del proyecto.

## Parámetros

| Parámetro | Descripción |
|-----------|-------------|
| **Suscripción** | Nombre o ID de la suscripción de Azure |
| **Grupo de recursos** | Grupo de recursos donde se encuentran el ACR y la Container App |
| **Nombre del ACR** | Nombre del Azure Container Registry |
| **Nombre de la imagen** | Nombre de la imagen Docker a construir |
| **Container App** | Nombre de la Azure Container App a actualizar |
| **Versión** | Tag de la imagen; se detecta automáticamente desde `__version__.py` si existe |

## Flujo de trabajo

### 1. Recopilación de parámetros

Se solicitan al usuario los datos de suscripción, grupo de recursos, ACR, nombre de imagen, Container App y versión. La versión se intenta detectar automáticamente buscando el archivo `__version__.py` en el workspace.

### 2. Configuración de suscripción

```bash
az account set --subscription "<suscripción>"
```

### 3. Resolución del login server del ACR

```bash
az acr show --name <acr> --resource-group <grupo_recursos> --query loginServer -o tsv
```

### 4. Build y push de la imagen

El skill intenta primero un **build local con Docker**. Si Docker no está disponible, recurre a un **build en la nube con ACR Tasks**.

- **Build local**: `docker build` → `docker tag` → `docker push`
- **Build en la nube**: `az acr build` (no requiere Docker instalado)

> Cuando se usa el build en la nube y el `Dockerfile` referencia una imagen base de Docker Hub, el skill verifica si la imagen ya existe en el ACR para evitar rate limits de Docker Hub. Si es necesario, la importa con `az acr import`.

### 5. Verificación de la imagen en ACR

```bash
az acr repository show-tags --name <acr> --repository <imagen> -o table
```

### 6. Actualización de la Container App

```bash
az containerapp update \
  --name <container_app> \
  --resource-group <grupo_recursos> \
  --image <login_server>/<imagen>:<versión>
```

### 7. Verificación del despliegue

Se ejecutan tres comprobaciones:

- La imagen en uso coincide con la versión desplegada.
- La revisión activa tiene `Active=True` y `TrafficWeight=100`.
- El tag de la imagen existe en el ACR.

### 8. Resumen final

Se presenta un resumen con la imagen, ACR, digest, Container App, revisión activa, réplicas, tráfico y estado del despliegue.

## Manejo de errores

- Si un comando `az` falla, se muestra el error con sugerencias (suscripción incorrecta, permisos, recurso no encontrado).
- Si Docker Hub aplica rate limit, se usa la estrategia de imagen base cacheada en el ACR.
- Si la actualización de la Container App falla, se verifica que las credenciales del ACR estén configuradas.
- Si el `Dockerfile` fue modificado temporalmente para el build en la nube, siempre se restaura al estado original.

## Licencia

Este proyecto es de uso interno de [PI Consulting](https://piconsulting.com.ar).
