# AKS - Azure Kubernetes Service

Infraestructura de Kubernetes en Azure usando **Terraform**. Provisiona un **AKS Cluster** completo con CNI, RBAC y Workload Identity.

## 📋 Descripción

Define y despliega un cluster AKS:
- **Kubernetes Cluster**: AKS gestionado en Azure
- **Node Pool**: Nodos para ejecutar pods (configurable)
- **Azure CNI**: Networking avanzado con IPs del subnet
- **RBAC**: Control de acceso basado en roles
- **Workload Identity**: Autenticación segura para pods

## 🏗️ Estructura

```
3. AKS/
├── aks.tf               # Recursos: Cluster AKS
├── data-network.tf      # Data sources (Network)
├── variables.tf         # Variables de entrada
├── outputs.tf           # Valores de salida
├── providers.tf         # Configuración de providers
└── README.md
```

## 🚀 Despliegue con Terraform

### Requisitos Previos
- Terraform 1.0+
- Azure CLI
- kubectl 1.20+
- Red (2. Network) ya desplegada

### Pasos

```bash
cd "3. AKS"
terraform fmt && terraform validate
terraform init
terraform plan -var-file="terraform.tfvars" -out="aks.tfplan"
terraform apply "aks.tfplan"
```

### Conectar kubectl

```bash
az aks get-credentials --resource-group rg-cloud-lab --name aks-e08
kubectl cluster-info
kubectl get nodes
```

## ✅ Validación del AKS

### Ver Nodos
```bash
kubectl get nodes -o wide
kubectl get nodes -o custom-columns=NAME:.metadata.name,IP:.status.addresses[0].address
```

### Validar desde Azure CLI
```bash
az aks show --resource-group rg-cloud-lab --name aks-e08
NODE_RG=$(az aks show -g rg-cloud-lab -n aks-e08 --query nodeResourceGroup -o tsv)
az network nic list --resource-group $NODE_RG --query "[].{IP:ipConfigurations[0].privateIPAddress}"
```

### Checklist
- ✅ Cluster accesible con kubectl
- ✅ Nodos en estado "Ready"
- ✅ IPs de nodos en `10.58.1.0/24`
- ✅ Pods del sistema corriendo

## 🚀 Orden Despliegue Correcto

1. ✅ **2. Network** ← Primero
2. ✅ **3. AKS** ← Segundo (ahora)
3. ✅ **1. App** ← Tercero

## 🚨 Destruir AKS (PRIMERO - OBLIGATORIO)

**ORDEN DESTRUCCIÓN (inverso):**
1. ✅ **3. AKS** ← Primero
2. ✅ **2. Network** ← Segundo
3. ✅ **1. App** ← Tercero

### Destruir AKS

```bash
cd "3. AKS"
terraform destroy -var-file="terraform.tfvars" -auto-approve
```

**Tiempo:** 5-10 minutos

### Verificar
```bash
kubectl cluster-info
# error: Unable to connect
```

**NUNCA intentes destruir Network si AKS aún existe.**

---

## 📚 Enlaces Útiles

- [Terraform Azure Provider - AKS](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/kubernetes_cluster)
- [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/es-es/azure/aks/)
- [Azure Container Networking Interface (CNI)](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni)
- [Workload Identity in AKS](https://docs.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [kubectl Documentation](https://kubernetes.io/docs/reference/kubectl/)
- [AKS Best Practices](https://docs.microsoft.com/en-us/azure/aks/best-practices)
- [AKS VM Size Selector](https://aka.ms/aks/vm-size-selector)

---

## 👤 Autor y Versión

- **Versión**: 1.0
- **Creado**: 2024-2025
- **Actualizado**: Febrero 2026
- **Propietario**: erick.iza
- **Equipo**: TCS Cloud Project

---

## 📄 Licencia

Este proyecto forma parte del laboratorio **TCS Cloud Project** - Laboratorio de Azure. Uso exclusivo para fines educativos y de demostración.

---

## 📞 Soporte

Para problemas o preguntas:
1. Validar conexión: `kubectl cluster-info`
2. Ver logs de eventos: `kubectl get events --all-namespaces`
3. Revisar estado del cluster: `az aks show --resource-group rg-cloud-lab --name aks-e08`
4. Consultar documentación oficial de AKS
5. Contactar al propietario del proyecto

---

**Última actualización**: 25 de febrero de 2026**
