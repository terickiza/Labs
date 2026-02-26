# ACR - Azure Container Registry

Infraestructura de registro de contenedores en Azure usando **Terraform**. Provisiona un **Azure Container Registry** completo con integración automática a AKS.

## 📋 Descripción

Define y despliega un registro privado de contenedores:
- **Container Registry**: Almacén privado de imágenes Docker
- **Admin User**: Usuario administrador para push/pull de imágenes
- **Public Access**: Acceso público configurado (solo el nombre, no las imágenes)
- **AKS Integration**: Integración automática con AKS mediante AcrPull role
- **Security**: Autenticación requerida para todas las operaciones

## 🏗️ Estructura

```
4. ACR/
├── main.tf              # Recursos: ACR + Integración AKS
├── variables.tf         # Variables de entrada
├── outputs.tf           # Valores de salida (credenciales, URLs)
├── versions.tf          # Versiones de providers
├── terraform.tfvars     # Valores de configuración
└── README.md
```

## 🚀 Despliegue con Terraform

### Requisitos Previos
- Terraform 1.0+
- Azure CLI
- Docker 20.10+
- AKS (3. AKS) ya desplegado

### Pasos

```bash
cd "4. ACR"
terraform fmt && terraform validate
terraform init
terraform plan -var-file="terraform.tfvars" -out="acr.tfplan"
terraform apply "acr.tfplan"
```

### Obtener Credenciales

```bash
# Ver login server
terraform output acr_login_server

# Ver usuario admin
terraform output -raw acr_admin_username

# Ver password admin
terraform output -raw acr_admin_password
```

## ✅ Validación del ACR

### Login al ACR
```bash
# Opción 1: Con Azure CLI (Recomendado)
az acr login --name <tu-acr-name>

# Opción 2: Con Docker
docker login <tu-acr-name>.azurecr.io
# Usuario: (del output acr_admin_username)
# Password: (del output acr_admin_password)
```

### Subir Imagen de Prueba
```bash
# Descargar imagen de prueba
docker pull nginx:latest

# Etiquetar con el nombre del ACR
docker tag nginx:latest <tu-acr-name>.azurecr.io/nginx:v1

# Subir al ACR
docker push <tu-acr-name>.azurecr.io/nginx:v1

# Verificar imagen subida
az acr repository list --name <tu-acr-name> --output table
```

### Validar desde Azure CLI
```bash
# Ver información del ACR
az acr show --name <tu-acr-name> --resource-group rg-cloud-lab --output table

# Listar todas las imágenes
az acr repository list --name <tu-acr-name> --output table

# Ver tags de una imagen específica
az acr repository show-tags --name <tu-acr-name> --repository nginx --output table

# Ver credenciales
az acr credential show --name <tu-acr-name> --output table
```

### Verificar Integración ACR-AKS
```bash
# Verificar que AKS puede descargar imágenes del ACR
az aks check-acr \
  --name aks-e08 \
  --resource-group rg-cloud-lab \
  --acr <tu-acr-name>.azurecr.io

# Debería mostrar: "ACR connection is successful"
```

### Checklist
- ✅ ACR accesible con `az acr login`
- ✅ Imágenes pueden subirse con `docker push`
- ✅ Integración con AKS funcional
- ✅ Credenciales obtenidas correctamente
- ✅ Login server responde

## 🚀 Orden Despliegue Correcto

1. ✅ **2. Network** ← Primero
2. ✅ **4. ACR** ← Segundo (ahora)
3. ✅ **3. AKS** ← Tercero
4. ✅ **1. App** ← Cuarto

## 🐳 Trabajar con Imágenes

### Construir y Subir Aplicación
```bash
# Construir imagen local
docker build -t <tu-acr-name>.azurecr.io/flask-app:v1 .

# Subir al ACR
docker push <tu-acr-name>.azurecr.io/flask-app:v1

# Verificar
az acr repository show-tags \
  --name <tu-acr-name> \
  --repository flask-app \
  --output table
```

### Descargar Imagen
```bash
# Descargar desde ACR
docker pull <tu-acr-name>.azurecr.io/flask-app:v1

# Listar imágenes locales
docker images | grep <tu-acr-name>
```

### Gestionar Imágenes
```bash
# Listar todos los repositorios
az acr repository list --name <tu-acr-name> --output table

# Ver detalles de un repositorio
az acr repository show \
  --name <tu-acr-name> \
  --repository flask-app

# Eliminar una imagen específica
az acr repository delete \
  --name <tu-acr-name> \
  --image flask-app:v1 \
  --yes
```

## 🚨 Destruir ACR (SEGUNDO - después de AKS)

**ORDEN DESTRUCCIÓN (inverso):**
1. ✅ **1. App** ← Primero (kubectl delete)
2. ✅ **3. AKS** ← Segundo
3. ✅ **4. ACR** ← Tercero (ahora)
4. ✅ **2. Network** ← Cuarto

### Destruir ACR

```bash
cd "4. ACR"
terraform destroy -var-file="terraform.tfvars" -auto-approve
```

**Tiempo:** 2-3 minutos

### Verificar
```bash
az acr show --name <tu-acr-name> --resource-group rg-cloud-lab
# (ResourceNotFound) The Resource ... was not found
```

**IMPORTANTE: Destruir AKS ANTES de destruir ACR para evitar errores de dependencias.**

---

## 📊 Configuración de SKUs

| SKU | Almacenamiento | Webhooks | Geo-replicación | Precio aprox. |
|-----|----------------|----------|-----------------|---------------|
| **Basic** | 10 GB | 2 | No | ~$5/mes |
| **Standard** | 100 GB | 10 | No | ~$20/mes |
| **Premium** | 500 GB | 500 | Sí | ~$50/mes |

**Para este proyecto**: Se usa **Basic** (suficiente para desarrollo y pruebas).

---

## 🔐 Seguridad

### Usuario Administrador
- **Uso**: Desarrollo y pruebas
- **Permisos**: Push, Pull, Delete
- **Rotación**: Cambiar passwords periódicamente

```bash
# Regenerar password
az acr credential renew \
  --name <tu-acr-name> \
  --password-name password
```

### Integración con AKS
- **Método**: Azure RBAC con AcrPull role
- **Ventaja**: No requiere credenciales en Kubernetes secrets
- **Seguridad**: Basado en managed identity del AKS

### Mejores Prácticas
- ✅ Usar managed identities en producción
- ✅ Deshabilitar admin user en producción
- ✅ Implementar Azure RBAC granular
- ✅ Escanear imágenes con Azure Security Center
- ✅ Implementar retention policies

---

## 📚 Enlaces Útiles

- [Terraform Azure Provider - ACR](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/container_registry)
- [Azure Container Registry Documentation](https://docs.microsoft.com/es-es/azure/container-registry/)
- [ACR Authentication](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication)
- [ACR Best Practices](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-best-practices)
- [Integrate ACR with AKS](https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration)
- [Docker Documentation](https://docs.docker.com/)
- [ACR Pricing](https://azure.microsoft.com/en-us/pricing/details/container-registry/)

---

## 👤 Autor y Versión

- **Versión**: 1.0
- **Creado**: Febrero 2026
- **Actualizado**: Febrero 2026
- **Propietario**: erick.iza
- **Equipo**: TCS Cloud Project

---

## 📄 Licencia

Este proyecto forma parte del laboratorio **TCS Cloud Project** - Laboratorio de Azure. Uso exclusivo para fines educativos y de demostración.

---

## 📞 Soporte

Para problemas o preguntas:
1. Validar conexión: `az acr login --name <tu-acr-name>`
2. Ver estado del ACR: `az acr show --name <tu-acr-name> --resource-group rg-cloud-lab`
3. Verificar imágenes: `az acr repository list --name <tu-acr-name>`
4. Revisar integración: `az aks check-acr --name aks-e08 --resource-group rg-cloud-lab --acr <tu-acr-name>.azurecr.io`
5. Consultar documentación oficial de ACR
6. Contactar al propietario del proyecto

---

## 🔍 Troubleshooting

### Error: "unauthorized: authentication required"
```bash
# Solución: Hacer login nuevamente
az acr login --name <tu-acr-name>
```

### Error: "denied: requested access to the resource is denied"
```bash
# Solución: Verificar permisos
az acr show --name <tu-acr-name> --query adminUserEnabled
# Si es 'false', habilitar admin user
az acr update --name <tu-acr-name> --admin-enabled true
```

### Error: "AKS no puede descargar imágenes del ACR"
```bash
# Solución: Verificar integración
az aks check-acr --name aks-e08 --resource-group rg-cloud-lab --acr <tu-acr-name>.azurecr.io

# Re-adjuntar ACR a AKS
az aks update --name aks-e08 --resource-group rg-cloud-lab --attach-acr <tu-acr-name>
```

### Error: "The registry name is already in use"
```bash
# Solución: El nombre del ACR debe ser único globalmente
# Cambiar el valor de 'acr_name' en terraform.tfvars
```

---

**Última actualización: 26 de febrero de 2026**
