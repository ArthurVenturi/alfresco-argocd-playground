# Repositório de Operações (GitOps) com Argo CD e Kustomize

Este repositório contém as configurações de infraestrutura como código (IaC) para o deploy de aplicações em ambientes Kubernetes, utilizando a metodologia GitOps com a ferramenta Argo CD.

## O que é GitOps?

GitOps é uma metodologia de entrega contínua (Continuous Delivery) para aplicações nativas da nuvem. Sua ideia central é utilizar um repositório Git como única fonte da verdade. Este repositório contém declarações descritivas da infraestrutura desejada, e um processo automatizado garante que o ambiente de produção corresponda ao estado descrito no Git.

O Argo CD é a ferramenta que faz a "ponte" entre o repositório Git e o cluster Kubernetes. Ele monitora o repositório e, assim que uma alteração é detectada, aplica-a automaticamente no cluster, garantindo que o estado real seja sempre igual ao estado desejado.

![Fluxo GitOps com Argo CD](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)
*Fonte: Documentação Oficial do Argo CD*

---

## Estrutura do Repositório com Kustomize

O repositório é organizado da seguinte forma para suportar múltiplos ambientes (`dev`, `hmg`, `prod`) e aplicações de forma clara e escalável.

```
/
├── _shared/
│   └── base/
│       ├── kustomization.yaml
│       └── ms-config-server-client.yaml  # Recursos compartilhados por todas as apps
│
├── apps/
│   └── minha-app/
│       ├── base/                         # Manifestos base da aplicação (comuns a todos os ambientes)
│       │   ├── kustomization.yaml
│       │   ├── deployment.yaml
│       │   └── service.yaml
│       │
│       └── overlays/                     # Configurações específicas por ambiente
│           ├── dev/
│           │   ├── kustomization.yaml
│           │   └── patch-replicas.yaml
│           │
│           └── prd/
│               ├── kustomization.yaml
│               └── patch-replicas.yaml
│
└── argo-cd-apps/                         # Manifestos das "Applications" do Argo CD
    ├── minha-app-dev.yaml
    └── minha-app-prd.yaml
```

- **`_shared/base`**: Contém recursos Kubernetes que são compartilhados entre todas as aplicações, como ConfigMaps, Secrets ou, no seu caso, a configuração do `ms-config-server-client`.
- **`apps/<nome-da-app>/base`**: Contém os manifestos Kubernetes base da aplicação (Deployment, Service, Ingress, etc.). Estes são os templates que não mudam entre ambientes.
- **`apps/<nome-da-app>/overlays/<ambiente>`**: Contém os patches do Kustomize para um ambiente específico (`dev`, `hmg`, `prod`). Aqui modificamos coisas como o número de réplicas, a tag da imagem, variáveis de ambiente, etc.
- **`argo-cd-apps/`**: Contém os manifestos `Application` do Argo CD. Cada arquivo aqui representa uma aplicação em um ambiente específico que o Argo CD deve gerenciar.

---

## Como Adicionar uma Nova Aplicação

Vamos seguir um exemplo para adicionar uma nova aplicação chamada `hello-world` no ambiente de `dev`.

### Passo 1: Criar os Manifestos Base

Primeiro, crie os arquivos de manifesto base para a sua aplicação.

1.  **Crie a estrutura de diretórios:**
    ```bash
    mkdir -p apps/hello-world/base
    ```

2.  **Crie o `deployment.yaml`:**
    `apps/hello-world/base/deployment.yaml`
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hello-world
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: hello-world
      template:
        metadata:
          labels:
            app: hello-world
        spec:
          containers:
          - name: hello-world
            image: nginx:1.21.6 # Usaremos uma imagem de exemplo
            ports:
            - containerPort: 80
    ```

3.  **Crie o `kustomization.yaml` da base:**
    `apps/hello-world/base/kustomization.yaml`
    ```yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
      - deployment.yaml
    ```

### Passo 2: Criar o Overlay de `dev`

Agora, vamos criar uma sobreposição (overlay) para o ambiente de desenvolvimento. Neste exemplo, vamos aumentar o número de réplicas para 2.

1.  **Crie a estrutura de diretórios:**
    ```bash
    mkdir -p apps/hello-world/overlays/dev
    ```

2.  **Crie o `kustomization.yaml` do overlay:**
    `apps/hello-world/overlays/dev/kustomization.yaml`
    ```yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    
    # Aponta para a base que queremos customizar
    bases:
      - ../../base
    
    # Define o número de réplicas para este ambiente
    replicas:
      - name: hello-world
        count: 2
    ```

### Passo 3: Criar a `Application` do Argo CD

Finalmente, crie o arquivo que instrui o Argo CD a monitorar e implantar nossa nova aplicação.

1.  **Crie o arquivo da aplicação:**
    `argo-cd-apps/hello-world-dev.yaml`
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: hello-world-dev
      namespace: argocd # Namespace onde o Argo CD está instalado
    spec:
      project: default
      source:
        repoURL: 'URL_DO_SEU_REPOSITORIO_GIT' # Substitua pela URL do seu repositório
        targetRevision: HEAD
        path: apps/hello-world/overlays/dev # Caminho para o Kustomize do ambiente
      destination:
        server: 'https://kubernetes.default.svc'
        namespace: dev # Namespace de destino no cluster Kubernetes
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
    ```

### Passo 4: Comitar e Sincronizar

1.  Adicione, comite e envie os novos arquivos para o seu repositório Git.
2.  O Argo CD detectará automaticamente a nova `Application` (se ele estiver configurado para monitorar o diretório `argo-cd-apps/`) ou você pode aplicá-la manualmente com `kubectl apply -f argo-cd-apps/hello-world-dev.yaml`.
3.  Após a `Application` ser criada, o Argo CD irá ler o caminho `apps/hello-world/overlays/dev`, gerar os manifestos com o Kustomize e aplicá-los no namespace `dev` do seu cluster.

Você poderá ver sua nova aplicação sincronizada na interface do Argo CD!

!Aplicações no Argo CD
*Fonte: Documentação Oficial do Argo CD*


## Guia Prático: Adicionando um novo serviço

Esta seção descreve o fluxo de trabalho para adicionar e configurar um novo serviço (ex: `app-ead-lms-manager`) em um ambiente (`dev`, `hmg` ou `prod`).

![](./assets/example-01.png)

1.  Na raiz do projeto, dentro do diretório `services`, crie uma nova pasta com o nome do seu serviço.

2.  Dentro deste novo diretório, adicione os manifestos Kubernetes essenciais: `deployment.json`, `service.json` e um `kustomization.yaml` que os agrupa. Você pode usar os serviços existentes como referência.

![](./assets/example-02.png)

### Passo 2: Habilitar o Serviço no Ambiente

Para que o Argo CD implante o serviço, adicione-o à lista de recursos do ambiente desejado.

*   Edite o arquivo `_shared/overlays/[ambiente]/resources/kustomization.yaml` e adicione o caminho para o diretório do seu novo serviço na seção `resources`.

![](./assets/example-03.png)


### Passo 3: Configurar Variáveis de Ambiente (ConfigMaps)

Para injetar configurações específicas do ambiente, como URLs de banco de dados, use `ConfigMaps`.

2.  Referencie este novo arquivo no `kustomization.yaml` do mesmo diretório (`confs`) para que ele seja aplicado.

![](./assets/example-04.png)


### Passo 4: Definir a Versão da Imagem

A tag da imagem do contêiner é definida por ambiente.

*   Edite o arquivo `_shared/overlays/[ambiente]/images/kustomization.yaml` e adicione ou modifique a entrada para o seu serviço, especificando a `newTag`.
    > **Atenção:** Pipelines de CI/CD frequentemente alteram este arquivo. Mantenha seu repositório local atualizado (`git pull`) antes de fazer modificações para evitar conflitos.

![](./assets/example-05.png)

### Passo 5: Definir o Número de Réplicas

*   Edite o arquivo `_shared/overlays/[ambiente]/replicas/kustomization.yaml` e adicione uma entrada para o seu serviço com a quantidade de réplicas (`count`).

![](./assets/example-06.png)


### Passo 6: Definir os recursos (services) que serão utilizados no ambiente

*   Edite o arquivo `_shared/overlays/[ambiente]/resources/kustomization.yaml` e adicione uma entrada para o seu serviço.

![](./assets/example-07.png)

### Passo 7: Configurar Volumes Persistentes (Opcional)

Se seu serviço precisa de armazenamento persistente, adicione os manifestos de `PersistentVolume` (PV) e `PersistentVolumeClaim` (PVC) no diretório `_shared/overlays/[ambiente]/volumes/` e referencie-os no `kustomization.yaml` correspondente.

![](./assets/example-08.png)

![](./assets/example-09.png)

![](./assets/example-10.png)

Após realizar as alterações, basta comitar e enviar para o repositório Git. O Argo CD cuidará da sincronização com o cluster.