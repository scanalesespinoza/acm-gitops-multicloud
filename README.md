# ACM GitOps Multicloud Base Repository

## Objetivo
Gestionar de un único punto la instalación y configuración de productos Red Hat (Ej. 3Scale) para que su despliegue y configuración se propague hacia todas las instancias habilitadas en los clusters manejados por Red Hat Advanced Cluster Management (ACM).

## Arquitectura

Esta solución integra **Red Hat Advanced Cluster Management (ACM)** y **Red Hat OpenShift GitOps (ArgoCD)** para habilitar la gestión unificada de configuraciones (GitOps) a escala multicluster.

### Flujo de Operación:
1. **Instalación de GitOps (Vía ACM Policy)**: Las políticas en el clúster Hub (`policies/install-gitops`) descubren los clústeres manejados y despliegan automáticamente el operador de OpenShift GitOps en ellos.
2. **Despliegue de Productos (Vía ApplicationSets)**: El propio Hub de GitOps utiliza *ApplicationSets* (`bootstrap/argo-cd`) integrados con ACM Placement para apuntar a los clústeres manejados y desplieggar los productos (ej. 3Scale).
3. **Sincronización de Configuraciones**: Los manifiestos base y parches específicos por entorno (mediante *Kustomize*, ubicados en `components/3scale`) son aplicados por el operador de GitOps local en el clúster manejado, garantizando que el estado final sea consistente con este repositorio Git.

## Estructura de Directorios

- `bootstrap/argo-cd/` - Configuraciones base de ApplicationSets en el Hub que despliegan productos en los clústeres remotos.
- `components/` - Bases de Kustomize para los diferentes productos de Red Hat (ej. `3scale`).
  - `components/<product>/base/` - Definiciones base (Namespace, OperatorGroup, Subscription, CustomResources).
  - `components/<product>/overlays/` - Parches y adaptaciones específicas para distintos entornos (dev, prod).
- `clusters/` - Configuraciones o parches específicos por clúster (ajustes finos).
- `policies/` - Políticas de ACM para el despliegue automático de la infraestructura base (ej. el operador de OpenShift GitOps).

---

## Configuración y Uso en ACM

Para orquestar todo este repositorio sobre una infraestructura multicluster con Red Hat ACM, sigue estos pasos explicados a continuación:

### 1. Prerrequisitos (En el Hub Cluster)
- Debes tener **Red Hat Advanced Cluster Management (ACM)** instalado y funcionando.
- Debes tener **OpenShift GitOps** instalado en el Hub (para alojar la instancia de ArgoCD central que actuará como controlador principal).
- Los clústeres remotos deben estar importados y bajo gestión (Managed) de ACM.

### 2. Etiquetado de Clústeres Administrados
Para que ACM sepa a qué clústeres debe enviar configuraciones o aplicaciones, debes añadir etiquetas o *labels* en la definición de cada `ManagedCluster`. 
- Etiqueta de ejemplo sugerida en este repo: `environment: prod` o `gitops: enabled`. (Validas esto según tu `PlacementRule`).

### 3. Aplicación de Gobernanza (Políticas y GitOps)
El primer paso práctico es lograr que todos los clústeres administrados (target) adquieran la capacidad GitOps y los objetos fundacionales.
- Aplica los recursos de la carpeta `policies/` desde el clúster Hub.
  ```bash
  oc apply -k policies/install-gitops/
  ```
- ACM distribuirá esta configuración hacia los clústeres administrados dictados por el `PlacementRule`. Esta política instalará el Operador de GitOps en el clúster remoto de forma automatizada comprobando que el estado final deseado se cumpla (*Enforce*).

### 4. Bootstrapping Inicial del Repositorio (ArgoCD en el Hub)
Una vez la infraestructura GitOps está operando en los nodos gestionados, debes enlazar este repositorio a tu ArgoCD central a través de una aplicación maestra (Patrón *App of Apps*) o aplicando directamente la carpeta de `bootstrap`.
- Aplica el "semillero" en el namespace del ArgoCD central en tu clúster Hub:
  ```bash
  oc apply -k bootstrap/argo-cd/ -n openshift-gitops
  ```
- Esto creará los `ApplicationSets` raíz. Estos ApplicationSets usarán generadores conectados a ACM (*ACM Cluster Decision Generator*).
- Todo clúster manejado que cumpla con los selectores del ApplicationSet recibirá instantáneamente y de forma automatizada las cargas de este repositorio (la instalación de 3scale, secretos, tenants y productos de APIs).

### 5. Configurar un Producto Nuevo o Actualizar Tenancy (Ejemplo Práctico)
Para agregar nuevos Tenants o Productos (APIs) a 3scale usando este modelo GitOps, debes operar siempre desde Git siguiendo la **Regla de Integración Continua** interna:

**Paso A: Aislar los cambios**
1. Crea una rama dedicada para tu nueva funcionalidad:
   ```bash
   git checkout -b feat/add-my-new-api
   ```

**Paso B: Definición Declarativa (Infrastructure as Code)**
2. Crea los manifiestos YAML (`Tenant`, `Product`, `Secret`) dentro del directorio base o los overlays específicos (ej. `components/3scale/overlays/prod/`).
   *Ejemplo de `Product` creado (`components/3scale/overlays/prod/echo-api-product.yaml`):*
   ```yaml
   apiVersion: capabilities.3scale.net/v1beta1
   kind: Product
   metadata:
     name: echo-api
   spec:
     name: "Echo API Product"
   ```
3. Registra de forma explícita los nuevos archivos creados dentro de la lista `resources:` del archivo `kustomization.yaml` ubicado en ese mismo directorio:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - ../../base
     - secrets.yaml
     - echo-api-product.yaml
   # ...
   ```

**Paso C: Sincronización y Orquestación**
4. Registra, empaqueta y sube tus cambios al repositorio remoto:
   ```bash
   git add components/3scale/overlays/prod/
   git commit -m "feat: Add new echo-api product and secrets to 3scale"
   git push origin feat/add-my-new-api
   ```
5. **Validación**: Abre un Pull Request (PR) en Github hacia tu rama principal o target (`main` / `production`).
6. Al fusionar el PR (Merge), el ArgoCD instalado en tu clúster Hub reaccionará al webhook (o en su próximo ciclo de sincronización) y empujará automáticamente la nueva configuración hacia todos los clústeres ACM designados. Ni tú ni el administrador necesitarán intervenir manualmente mediante `oc apply` en el ecosistema 3scale interactivo.
