# Desafio de Kubernetes - CloudOps Bootcamp

Repositório do desafio da S7 (Kubernetes) do CloudOps Bootcamp. A proposta era
implantar uma API integrada a um banco de dados dentro de um cluster local, e
provar na prática que os dados sobrevivem mesmo quando o Pod do banco morre e
é recriado.

## O que foi implantado

- **PostgreSQL** rodando com volume persistente (PVC)
- **PostgREST** como API, conectada ao Postgres via Service (DNS interno do
  cluster, sem IP fixo)
- Credenciais do banco em Secret, configs não sensíveis em ConfigMap
- Health checks (liveness/readiness) e requests/limits na API
- HPA configurado como bônus

A ideia central do desafio é essa: a API acha o banco pelo nome do Service, não
pelo IP do Pod. Isso importa porque o IP do Pod muda toda vez que ele é
recriado, mas o nome do Service não muda nunca.

## Arquitetura

```
curl / navegador
      |
      v
  Service (API)
      |
      v
Deployment PostgREST  --- conexão via Service DNS --->  Deployment PostgreSQL + PVC
```

A API não sabe o IP do Postgres, só sabe o nome do Service
(`postgres-service`, por exemplo). O Kubernetes resolve esse nome pra IP
internamente. Se o Pod do banco morrer e subir outro no lugar, o IP muda mas
o nome do Service continua o mesmo, então a API nem percebe a troca.

## Ferramentas usadas

- WSL2 (Ubuntu) rodando no Windows
- Minikube (driver docker)
- kubectl
- Docker Desktop com integração WSL

## Pré-requisitos

- Docker instalado e rodando (Docker Desktop com integração WSL, no meu caso)
- Minikube instalado
- kubectl instalado
- WSL2 configurado, se for Windows

## Antes de rodar

Precisa ter o cluster local de pé:

```bash
minikube start --driver=docker
kubectl get nodes
```

Espera o node aparecer como `Ready` antes de continuar.

## Estrutura do repositório

```
manifests/       -> todos os YAMLs, numerados na ordem de aplicação
evidencias/      -> prints separados por nível do desafio
README.md
```

## Subindo tudo

```bash
kubectl apply -f manifests/
```

Se preferir ir aplicando um por um pra acompanhar o que cada recurso faz:

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl apply -f manifests/01-configmap.yaml
kubectl apply -f manifests/02-secret.yaml
kubectl apply -f manifests/03-postgres-pvc.yaml
kubectl apply -f manifests/04-postgres-deployment.yaml
kubectl apply -f manifests/05-postgres-service.yaml
kubectl apply -f manifests/06-postgrest-deployment.yaml
kubectl apply -f manifests/07-postgrest-service.yaml
```

## Sobre o Secret

O arquivo de Secret que está no repositório é só o template/estrutura. A
senha real não vai commitada. Pra gerar o seu:

```bash
kubectl create secret generic postgres-secret \
  --namespace=desafio-kubernetes \
  --from-literal=POSTGRES_USER=usuario \
  --from-literal=POSTGRES_PASSWORD=sua-senha \
  --dry-run=client -o yaml > manifests/02-secret.yaml
```

## Acessando a API

```bash
kubectl port-forward -n desafio-kubernetes svc/postgrest-service 3000:3000
```

Com isso rodando, a API responde em `http://localhost:3000`.

Inserindo um dado:

```bash
curl -X POST http://localhost:3000/NOME_DA_TABELA \
  -H "Content-Type: application/json" \
  -d '{"campo": "valor"}'
```

Consultando:

```bash
curl http://localhost:3000/NOME_DA_TABELA
```

## Provando que os dados persistem

Esse é o ponto principal do desafio. O teste que eu fiz:

1. Insere um dado via POST
2. Confirma com GET que ele está lá
3. Descobre o nome do Pod do Postgres: `kubectl get pods -n desafio-kubernetes`
4. Deleta esse Pod: `kubectl delete pod <nome-do-pod> -n desafio-kubernetes`
5. Espera o Deployment subir um Pod novo no lugar
6. Faz o GET de novo

O dado continua lá porque ele está gravado no PVC, não no Pod. O Pod é
descartável, o volume não.

## Verificando as réplicas e os probes

```bash
kubectl get pods -n desafio-kubernetes -l app=postgrest
kubectl describe pod <nome-do-pod-api> -n desafio-kubernetes
```

No describe dá pra ver as seções de Liveness e Readiness configuradas.

## Limpando tudo

```bash
kubectl delete namespace desafio-kubernetes
```

Remove tudo que foi criado de uma vez, sem precisar deletar recurso por
recurso.

## Evidências

Os prints de cada etapa estão em `evidencias/`, separados por nível
(nivel1 até nivel7), seguindo os critérios de aceitação do desafio.