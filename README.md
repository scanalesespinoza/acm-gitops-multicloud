# ACM GitOps Multicloud Base Repository

## Objetivo
Gestionar de un único punto la instalación y configuración de productos Red Hat (Ej. 3Scale) para que su despliegue y configuración se propague hacia todas las instancias habilitadas en los clusters manejados por Red Hat Advanced Cluster Management (ACM).

## Arquitectura

Esta solución integra **Red Hat Advanced Cluster Management (ACM)** y **Red Hat OpenShift GitOps (ArgoCD)** para habilitar la gestión unificada de configuraciones (GitOps) a escala multicluster.

### Flujo de Operación:
1. **Instalación de GitOps (Vía ACM Policy)**: Las políticas en el clúster Hub (`policies/install-gitops`) descubren los clústeres manejados y despliegan automáticamente el operador de OpenShift GitOps en ellos.
2. **Despliegue de Productos (Vía ApplicationSets)**: El propio Hub de GitOps utiliza *ApplicationSets* (`bootstrap/argo-cd`) integrados con ACM Placement para apuntar a los clústeres manejados y desplegar los productos (ej. 3Scale).
3. **Sincronización de Configuraciones**: Los manifiestos base y parches específicos por entorno (mediante *Kustomize*, ubicados en `components/3scale`) son aplicados por el operador de GitOps local en el clúster manejado, garantizando que el estado final sea consistente con este repositorio Git.

## Estructura de Directorios

- `bootstrap/argo-cd/` - Configuraciones base de ApplicationSets en el Hub que despliegan productos en los clústeres remotos.
- `components/` - Bases de Kustomize para los diferentes productos de Red Hat (ej. `3scale`).
  - `components/<product>/base/` - Definiciones base (Namespace, OperatorGroup, Subscription, CustomResources).
  - `components/<product>/overlays/` - Parches y adaptaciones específicas para distintos entornos (dev, prod).
- `clusters/` - Configuraciones o parches específicos por clúster (ajustes finos).
- `policies/` - Políticas de ACM para el despliegue automático de la infraestructura base (ej. el operador de OpenShift GitOps).

## Uso

1. El servidor ArgoCD del Hub debe estar configurado para reconciliar contra este repositorio.
2. Para habilitar un producto como 3Scale en nuevos clústeres manejados, basta con etiquetar el clúster en ACM para que cumpla con los selectores de los ApplicationSets creados en `bootstrap/argo-cd/`.
