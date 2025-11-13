# Provisionamento do Wazuh no OpenShift

Este guia descreve como preparar um projeto dedicado no OpenShift para hospedar o Wazuh, garantindo limites de recursos, contas de serviço e armazenamento persistente adequados.

## 1. Criar o projeto e configurar quotas/limites

```bash
oc new-project wazuh
```

Em seguida, aplique uma `ResourceQuota` para limitar o consumo agregado do projeto e um `LimitRange` para definir requests/limits padrão por contêiner:

```bash
cat <<'YAML' | oc apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: wazuh-quota
  namespace: wazuh
spec:
  hard:
    pods: "30"
    requests.cpu: "6"
    requests.memory: 12Gi
    limits.cpu: "12"
    limits.memory: 24Gi
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: wazuh-default-limits
  namespace: wazuh
spec:
  limits:
    - type: Container
      default:
        cpu: "1000m"
        memory: 2Gi
      defaultRequest:
        cpu: "250m"
        memory: 512Mi
      max:
        cpu: "4000m"
        memory: 8Gi
      min:
        cpu: "100m"
        memory: 256Mi
YAML
```

Ajuste os valores conforme a capacidade do cluster e a carga esperada.

## 2. ServiceAccounts e permissões

Crie contas de serviço dedicadas para cada componente do Wazuh e vincule as permissões necessárias:

```bash
oc create serviceaccount wazuh-manager -n wazuh
oc create serviceaccount wazuh-indexer -n wazuh
oc create serviceaccount wazuh-dashboard -n wazuh
```

Para permitir acesso a recursos de monitoramento, configuração e volumes persistentes, utilize um `ClusterRole` personalizado e os `RoleBindings` correspondentes:

```bash
cat <<'YAML' | oc apply -f -
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: wazuh-components
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints", "configmaps", "persistentvolumeclaims", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: wazuh-manager-binding
  namespace: wazuh
subjects:
  - kind: ServiceAccount
    name: wazuh-manager
    namespace: wazuh
roleRef:
  kind: ClusterRole
  name: wazuh-components
  apiGroup: rbac.authorization.k8s.io
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: wazuh-indexer-binding
  namespace: wazuh
subjects:
  - kind: ServiceAccount
    name: wazuh-indexer
    namespace: wazuh
roleRef:
  kind: ClusterRole
  name: wazuh-components
  apiGroup: rbac.authorization.k8s.io
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: wazuh-dashboard-binding
  namespace: wazuh
subjects:
  - kind: ServiceAccount
    name: wazuh-dashboard
    namespace: wazuh
roleRef:
  kind: ClusterRole
  name: wazuh-components
  apiGroup: rbac.authorization.k8s.io
YAML
```

Se algum componente necessitar de privilégios adicionais (por exemplo, para montar volumes com `SecurityContextConstraints` específicos), crie `RoleBindings` adicionais apontando para as SCCs apropriadas.

## 3. Armazenamento persistente

O Wazuh exige volumes persistentes distintos para o manager, o indexer (cluster OpenSearch) e os dashboards. Valide com a equipe de infraestrutura que os `StorageClasses` disponíveis suportam os requisitos abaixo:

| Componente | Tamanho mínimo sugerido | Acesso | Observações |
|------------|------------------------|--------|-------------|
| Manager    | 100 Gi                 | ReadWriteOnce | Armazena logs e dados de agentes. Preferir discos SSD para melhor I/O. |
| Indexer    | 250 Gi por nó          | ReadWriteOnce | Utilizado pelo OpenSearch. Necessita suporte a snapshots ou expansão online, se possível. |
| Dashboards | 20 Gi                  | ReadWriteOnce | Persistência de configurações e histórico de visualizações. |

Crie os `PersistentVolumeClaims` garantindo os requests compatíveis com as quotas configuradas:

```bash
cat <<'YAML' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wazuh-manager-data
  namespace: wazuh
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wazuh-indexer-data
  namespace: wazuh
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 250Gi
  storageClassName: fast-ssd
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wazuh-dashboard-data
  namespace: wazuh
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd
YAML
```

Substitua o `storageClassName` pelos valores disponíveis no cluster. Utilize `oc describe storageclass` para listar as classes e `oc get pvc -n wazuh` para confirmar o status das requisições.

## 4. Próximos passos

Com o projeto, quotas, permissões e armazenamento definidos, prossiga com a implantação dos charts/manifests do Wazuh, garantindo que cada `Deployment` ou `StatefulSet` utilize as `ServiceAccounts` e os `PersistentVolumeClaims` configurados acima.

