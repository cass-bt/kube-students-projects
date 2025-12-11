# Projeto Kubernetes - Deploy de Aplicação Fullstack

## 👥 Integrantes da Equipe
| Nome | Matrícula |
|------|-----------|
| Nome do Aluno 1 | 000000 |
| Nome do Aluno 2 | 000000 |
| Nome do Aluno 3 | 000000 |

## 🎯 Objetivo do Projeto
Realizar o deploy completo de uma aplicação fullstack (React + Flask + PostgreSQL) em um cluster Kubernetes local (Kind), garantindo:
- Alta disponibilidade com réplicas para frontend e backend
- Persistência de dados utilizando PersistentVolumeClaim
- Configuração via ConfigMap e Secrets
- Acesso externo via NGINX Ingress Controller

## 🏗️ Arquitetura da Aplicação

```
                    ┌─────────────────────────────────────────────────────┐
                    │                   INGRESS NGINX                      │
                    │         http://localhost ou domínio                  │
                    └──────────────┬────────────────┬─────────────────────┘
                                   │                │
                          /        │                │  /api
                                   ▼                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         NAMESPACE: app                                    │
│  ┌─────────────────────────┐      ┌─────────────────────────┐            │
│  │   Frontend (React)      │      │   Backend (Flask)       │            │
│  │   Deployment: 2 réplicas│      │   Deployment: 2 réplicas│            │
│  │   Service: ClusterIP    │      │   Service: ClusterIP    │            │
│  │   Port: 80              │      │   Port: 5000            │            │
│  └─────────────────────────┘      └────────────┬────────────┘            │
│                                                 │                         │
│         ConfigMap: frontend-config              │  ConfigMap: backend-config
│                                                 │  Secret: db-credentials │
└─────────────────────────────────────────────────┼─────────────────────────┘
                                                  │
                                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         NAMESPACE: database                               │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                     PostgreSQL                                       │ │
│  │                     StatefulSet: 1 réplica                          │ │
│  │                     Service: ClusterIP (port 5432)                  │ │
│  │                     PVC: 1Gi                                        │ │
│  │                     Secret: postgres-secret                         │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

## 📁 Estrutura do Projeto

```
projeto-k8s-deploy/
├── README.md                        # Este arquivo
├── namespace.yaml                   # Namespaces: app e database
├── frontend/
│   └── deployment.yaml              # Deployment + Service + ConfigMap do React
├── backend/
│   ├── deployment.yaml              # Deployment + Service do Flask
│   └── configmap.yaml               # ConfigMap com variáveis de ambiente
├── database/
│   ├── statefulset.yaml             # StatefulSet + Service do PostgreSQL
│   ├── pvc.yaml                     # PersistentVolumeClaim
│   └── secret.yaml                  # Secret com credenciais
└── ingress/
    └── ingress.yaml                 # IngressClass + Ingress rules
```

## 🚀 Pré-requisitos

1. **Docker** instalado e funcionando
2. **Kind** (Kubernetes in Docker) instalado
3. **kubectl** configurado
4. **Imagens Docker** publicadas no DockerHub

### Criar cluster Kind com Ingress

```bash
# Criar arquivo de configuração do Kind
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

### Instalar NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Aguardar o Ingress Controller estar pronto
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

## 📦 Deploy da Aplicação

### Passo 1: Criar os Namespaces

```bash
kubectl apply -f namespace.yaml
```

### Passo 2: Deploy do Banco de Dados (namespace: database)

```bash
# Aplicar Secret com credenciais
kubectl apply -f database/secret.yaml

# Criar PersistentVolumeClaim
kubectl apply -f database/pvc.yaml

# Deploy do PostgreSQL
kubectl apply -f database/statefulset.yaml
```

### Passo 3: Deploy do Backend (namespace: app)

```bash
# Aplicar ConfigMap
kubectl apply -f backend/configmap.yaml

# Deploy do Backend Flask
kubectl apply -f backend/deployment.yaml
```

### Passo 4: Deploy do Frontend (namespace: app)

```bash
kubectl apply -f frontend/deployment.yaml
```

### Passo 5: Configurar Ingress

```bash
kubectl apply -f ingress/ingress.yaml
```

### Deploy Completo (Alternativo)

```bash
# Aplicar todos os recursos de uma vez
kubectl apply -f namespace.yaml
kubectl apply -f database/
kubectl apply -f backend/
kubectl apply -f frontend/
kubectl apply -f ingress/
```

## ✅ Verificação do Deploy

### Verificar Namespaces

```bash
kubectl get namespaces
```

### Verificar Pods

```bash
# Pods do namespace app
kubectl get pods -n app

# Pods do namespace database
kubectl get pods -n database

# Todos os pods
kubectl get pods --all-namespaces
```

### Verificar Services

```bash
kubectl get svc -n app
kubectl get svc -n database
```

### Verificar Ingress

```bash
kubectl get ingress -n app
kubectl describe ingress app-ingress -n app
```

### Verificar PVC

```bash
kubectl get pvc -n database
```

### Logs dos Pods

```bash
# Logs do backend
kubectl logs -l app=backend -n app

# Logs do frontend
kubectl logs -l app=frontend -n app

# Logs do PostgreSQL
kubectl logs -l app=postgres -n database
```

## 🌐 Acesso à Aplicação

Após o deploy, a aplicação estará disponível em:

| Componente | URL |
|------------|-----|
| Frontend | http://localhost/ |
| Backend API | http://localhost/api/ |

### Testar Backend via curl

```bash
# Health check
curl http://localhost/api/health

# Listar mensagens
curl http://localhost/api/messages

# Criar mensagem
curl -X POST http://localhost/api/messages \
  -H "Content-Type: application/json" \
  -d '{"content": "Hello Kubernetes!"}'
```

## 🔧 Comandos Úteis

### Port-Forward para Debug

```bash
# Acessar backend diretamente
kubectl port-forward svc/backend-service 5000:5000 -n app

# Acessar PostgreSQL diretamente
kubectl port-forward svc/postgres 5432:5432 -n database
```

### Escalar Réplicas

```bash
# Escalar frontend
kubectl scale deployment frontend --replicas=3 -n app

# Escalar backend
kubectl scale deployment backend --replicas=3 -n app
```

### Reiniciar Deployments

```bash
kubectl rollout restart deployment/frontend -n app
kubectl rollout restart deployment/backend -n app
```

## 🗑️ Limpeza

```bash
# Remover todos os recursos
kubectl delete -f ingress/
kubectl delete -f frontend/
kubectl delete -f backend/
kubectl delete -f database/
kubectl delete -f namespace.yaml

# Ou deletar os namespaces (remove tudo dentro)
kubectl delete namespace app database
```

## 📝 Observações Importantes

1. **Imagens Docker**: Antes do deploy, substitua `<SEU_DOCKERHUB_USER>` nos arquivos de deployment pela sua conta do DockerHub.

2. **Credenciais**: Os valores no Secret estão em base64. Para gerar novos valores:
   ```bash
   echo -n 'seu_valor' | base64
   ```

3. **Persistência**: Os dados do PostgreSQL são persistidos no PVC. Mesmo reiniciando o cluster, os dados são mantidos.

4. **Probes**: Os deployments incluem `livenessProbe` e `readinessProbe` para garantir a saúde dos containers.

## 🏆 Recursos Bonus Implementados

- ✅ Liveness e Readiness Probes em todos os componentes
- ✅ Resource limits e requests
- ✅ Labels organizacionais
- ✅ Namespaces separados para isolamento

## 📚 Referências

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kind - Kubernetes in Docker](https://kind.sigs.k8s.io/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
