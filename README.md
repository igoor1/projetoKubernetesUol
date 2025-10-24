![Rancher](https://img.shields.io/badge/rancher-%230075A8.svg?style=for-the-badge&logo=rancher&logoColor=white)
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326CE5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-%23EF7422.svg?style=for-the-badge&logo=argo&logoColor=white)
![GitHub](https://img.shields.io/badge/github-%23181717.svg?style=for-the-badge&logo=github&logoColor=white)

# Projeto Kubernetes com GitOps na prática - Online Boutique

## Sobre o Projeto

Este projeto demonstra a implementação de **GitOps** utilizando ArgoCD, Kubernetes e GitHub. 
O objetivo é executar um conjunto de microserviços (Online Boutique) em um cluster Kubernetes local, com o estado do cluster sendo gerenciado via GitOps pelo ArgoCD a partir de um repositório no GitHub.

### Índice 
1. [Pré-requisitos](#1-pré-requisitos)
2. [Preparação do Ambiente](#2-preparação-do-ambiente)
    - 2.1. [Instalação do Rancher Desktop](#21-instalação-do-rancher-desktop)
    - 2.2. [Instalação do ArgoCD no cluster local](#22-instalação-do-argocd-no-cluster-local)
3. [Fork e repositório GitHub](#3-fork-e-repositório-github)
4. [Criando aplicação no ArgoCD](#4-criando-aplicação-no-argocd)
5. [Acessando o Front-end](#5-acessando-o-front-end)
6. [O Fluxo GitOps em Ação (Exemplo de Scaling)](#6-o-fluxo-gitops-em-ação-exemplo-de-scaling)
7. [Desafio extra: Conectar a um Repositório Privado](#7-desafio-extra-conectar-a-um-repositório-privado)
8. [Referências](#8-referências)


## 1. Pré-requisitos

- Conta no GitHub.
- Conhecimento básico de Git, Docker e Kubernetes.
- Git instalado [Download Git](https://git-scm.com/downloads)
- WSL (Windows Subsystem for Linux). [Guia de instalação do WSL2](https://learn.microsoft.com/pt-br/windows/wsl/install)

## 2. Preparação do Ambiente

### 2.1. Instalação do Rancher Desktop

1. Baixe e instale o Rancher Desktop: https://rancherdesktop.io/

2. Durante a instalação ou na primeira execução, escolha **dockerd (moby)** como o container runtime e **habilite o Kubernetes**.

![DockerD](https://github.com/user-attachments/assets/c8bbc2ef-91c3-4181-a39b-81f411cc174e)

3. Após a instalação ser concluída, verifique se o node do seu cluster local está com o status Ready.

```bash
kubectl get nodes
```

### 2.2. Instalação do ArgoCD no cluster local

1. Crie um namespace para o ArgoCD:

```bash
kubectl create namespace argocd
```

2. Aplique o manifesto de instalação do ArgoCD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

3. Para acessar a interface web (UI) do ArgoCD, abra um novo terminal e faça o encaminhamento da porta de serviço:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

4. Retorne ao prompt anterior e execute o comando no Git Bash para obter senha inicial gerada pelo ArgoCD:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

![Get Password](https://github.com/user-attachments/assets/6ccd99cb-141a-4647-b732-82ce0d7bdc49)

5. Acesse a UI do ArgoCD no seu navegador: https://localhost:8080

- **login:** `admin`
- **senha:** `utilize a senha decodificada anteriormente.`

![LoginPage ArgoCD](https://github.com/user-attachments/assets/036e2dc0-2a3e-45a1-b5d9-8f5e1eaed706)


## 3. Fork e repositório GitHub

Este repositório Git será a "fonte da verdade" para o nosso cluster. O ArgoCD irá monitorá-lo constantemente e garantir que o estado do cluster seja um reflexo exato do que está definido nos manifestos deste repositório.
Para este projeto, usaremos a aplicação de exemplo "Online Boutique" da Google.

1. **Faça um Fork do Repositório:** Vá até o repositório oficial da aplicação e clique em "Fork" para criar uma cópia na sua própria conta do GitHub. https://github.com/GoogleCloudPlatform/microservices-demo

2. Crie um novo repositorio no GitHub:

![new repository](https://github.com/user-attachments/assets/e62b4d18-a7b9-4a96-9603-d2d0467c2c16)

> [!IMPORTANT]
> Para este guia, estamos usando um repositório público.

3. Clone este repositório vazio.

4. Copie o arquivo `kubernetes-manifests.yaml` do [link](https://github.com/GoogleCloudPlatform/microservices-demo/blob/main/release/kubernetes-manifests.yaml).

5. Cole-o como `K8s/online-boutique.yaml` dentro do seu repositório.

**Estrutura recomendada do seu repositório:**
```
SeuProjeto/
└── K8s
  └── online-boutique.yaml
```

6. Faça o commit e push.

## 4. Criando aplicação no ArgoCD

Agora, vamos instruir o ArgoCD a monitorar nosso repositório e aplicar os manifestos no cluster.

1. No ArgoCD, clique em **+ NEW APP**

2. Preencha os campos da seguinte forma: 

- **Application Name:** online-boutique

- **Project Name:** default

- **Sync Policy:** Manual (podemos mudar para Automatic depois)

- **Repository URL:** Cole a URL do seu fork do repositório.

- **Revision**: HEAD

- **Path:** Caminho para a pasta que contém os manifestos Kubernetes

- **Cluster URL:** https://kubernetes.default.svc (deve ser o valor padrão para o cluster local)

- **Namespace:** default

![new app](https://github.com/user-attachments/assets/b243c6eb-6435-4976-8b44-f711a7755c50)
![new app 2](https://github.com/user-attachments/assets/19f31e86-b395-4fe2-a6ab-a62d73226661)

3. Clique em **CREATE**.

O ArgoCD irá criar a aplicação e mostrará o status **Missing** e **OutOfSync**, o que é normal. Isso significa que os recursos definidos no Git ainda não existem no cluster.

4. Com a aplicação criada, clique em **SYNC**.

5. Mantenha as opções padrão e clique em **SYNCHRONIZE**.

> **Nota:** O processo de sincronização pode levar alguns minutos enquanto o Kubernetes baixa as imagens dos containers e inicializa todos os pods.

![synced](https://github.com/user-attachments/assets/ddc7847b-282d-4d1e-9f4b-d7a47d16b04f)

6. Para verificar se os pods estão rodando, você pode usar o kubectl:

```bash
kubectl get pods
```

## 5. Acessando o Front-end
Após o ArgoCD sincronizar a aplicação, todos os pods e services estarão rodando no seu cluster. Para acessar a loja virtual (o serviço de frontend) do seu navegador, precisamos expor a porta do serviço para a sua máquina local.

1. Abra um novo terminal (deixe os outros rodando) e execute o comando port-forward:

```bash
kubectl port-forward svc/frontend-external 8081:80
```

2. Abra seu navegador e acesse: `http://localhost:8081`.

3. Você deverá ver a página inicial da aplicação "Online Boutique".

![home](https://github.com/user-attachments/assets/385fea57-574f-4f9e-8226-19b38db95817)
![sunglass](https://github.com/user-attachments/assets/f3cd10b4-3dbf-4595-a52a-d63509e4129d)


## 6. O Fluxo GitOps em Ação (Exemplo de Scaling)

Este é o conceito mais importante do GitOps. O repositório Git é a única fonte da verdade. Todas as alterações no cluster devem ser feitas através de um commit no Git, e não com comandos manuais.

1. Vá até o seu repositório no GitHub.

2. Navegue até o seu manifesto `online-boutique.yaml` e edite-o.

3. Procure pelo `Deployment` chamado `frontend`, e altere o campo replicas de `1` para `3`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3  # Alterado de 1 para 3
```

![replicas](https://github.com/user-attachments/assets/d4934ac3-1643-4566-8138-42ccf68b03e1)

4. Faça o commit da alteração diretamente no GitHub.

5. No ArgoCD, se você habilitou o **"Auto-Sync"**, dentro de 3 minutos ele aplicará a mudança sozinho. Se não, clique em **SYNC** para aplicar a alteração manualmente.

6. Após a sincronização, verifique as réplicas no terminal:

```bash
kubectl get deployments
```

![replica result](https://github.com/user-attachments/assets/817982bd-3b33-4c3e-9772-c56094699b8d)


## 7. Desafio extra: Conectar a um Repositório Privado 

Para que o ArgoCD possa ler um repositório Git privado, você precisa fornecer a ele credenciais de acesso, preferencialmente via Personal Access Token (PAT).

1. No GitHub:
- Acesse Settings > Developer settings > Personal access tokens > Tokens (classic)
![GitHub token](https://github.com/user-attachments/assets/ff2cc968-3e9c-4eb6-8004-24fa6cad03ee) 

- Clique em Generate new token.
- Dê um nome (ex: argocd-token) e marque a permissão repo (Controle total de repositórios privados).
- Gere o token e copie o valor imediatamente. Você não poderá vê-lo novamente.

2. No ArgoCD:
- Navegue até Settings > Repositories.
- Clique em CONNECT REPO.

![Connect repo](https://github.com/user-attachments/assets/612fb51c-a512-4cfd-8c13-2466e0e10a47) 

- Preencha os campos com as informações do seu repositório privado:

![private Repo](https://github.com/user-attachments/assets/62a366a4-0a95-400f-aea3-af5384fd931e)
- Clique em CONNECT.

3. Resultado

- Se tudo estiver correto, você verá a conexão com o status Successful. Agora você pode criar aplicações usando este repositório privado.
![Connect Result](https://github.com/user-attachments/assets/c8ed803e-9149-4077-a866-62c0bb451300)


## 8. Referências

* [ArgoCD - Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started)
* [ArgoCD - Private Repositories](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories)
* [Rancher Desktop](https://www.rancher.com/products/rancher-desktop)
* [Aplicação de Exemplo - Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo)
