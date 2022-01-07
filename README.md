<html>
  <body>
    <h1> Implementação do ARGOCD com Kustomize </h1>
   
   <h3> Introdução </h3>
 
   > Tutorial e incentivo de como implementar uma stack do Argocd no kubernetes e gitlab.
    
   <h3> Recomendações </h3>
    
   >   1. Argocd CLI instalado.
   >   2. Cluster Kubernetes ativo.
    
   <h3>  Organização de Projetos </h3>
    
   >   Boas práticas recomendam a utilização de dois projetos na implementação de uma stack do argocd:
   >   1. Um projeto onde será armazenado todos os arquivos de manifesto das aplicações (deployments, services, ingress) que é onde o ArgoCD irá monitorar.
   >   2. E o segundo projeto, que seria o projeto da aplicação em si, onde mudanças deployadas seriam refletidas no primeiro projeto, normalmente modificando imagem.
    
   <h3> Organização de pastas </h3>
    
   >  O ArgoCD é uma ferramenta que é integrada ao GitLab, onde monitora possíveis mudanças em branchs específicas ou a partir de commits específicos, portanto requer uma organização bem estrutura para se conseguir realizar um monitoramente adequado.
   
   >  A estrutura é dividida em 2 partes dentro do gitlab:
    
   >  Repositório dos arquivos de configuração do argoCD (applications e projects).
   >  1. Repositórios separados por ambiente. (Produção, Homologação e testes).
    
  ```sh
        argo
        |__ prod
          |__ namespace-prod
                |__ argo-namespace-project.yaml
                |__ aplicacao1-prod
                  |__ argo-aplicacao1-application.yaml
                |__ aplcacao2-prod
                  |__ argo-aplicacao1-application.yaml
        |__ hml
          |__ namespace-hml
                |__ argo-namespace-project.yaml
                |__ aplicacao1-hml
                  |__ argo-aplicacao1-application.yaml
                |__ aplcacao2-hml
                  |__ argo-aplicacao1-application.yaml
        |__ qa
          |__ namespace-qa
                |__ argo-namespace-project.yaml
                |__ aplicacao1-qa
                  |__ argo-aplicacao1-application.yaml
                |__ aplcacao2-qa
                  |__ argo-aplicacao1-application.yaml
   ```
   >  2. Repositório de deployments do kubernetes (deployments, ingress, services e arquivo kustomization).
      Dentro da estrutura tem duas pastas: bases e overlays.
    
   ```sh
         Deployments
          |__ prod/
              |__ namespace/
                  |__ aplicacao/
                      |__ bases/
                          |__ aplicacao-deployment.yaml
                          |__ aplicacao-service.yaml
                          |__ aplicacao-ingress.yaml
                          |__ kustomization.yaml
                      |__ overlays/ # Pasta que o argo irá monitorar
                          |__ env-prod.yaml  # Se necessário modificar variáveis de ambiente.
                          |__ kustomization.yaml
          |__ hml/
              |__ namespace/
                  |__ aplicacao/
                      |__ bases/
                          |__ aplicacao-deployment.yaml
                          |__ aplicacao-service.yaml
                          |__ aplicacao-ingress.yaml
                          |__ kustomization.yaml
                      |__ overlays/ # Pasta que o argo irá monitorar
                          |__ env-hml.yaml  # Se necessário modificar variáveis de ambiente.
                          |__ kustomization.yaml
            |__ qa
               |__ namespace/
                  |__ aplicacao/
                      |__ bases/
                          |__ aplicacao-deployment.yaml
                          |__ aplicacao-service.yaml
                          |__ aplicacao-ingress.yaml
                          |__ kustomization.yaml  
                      |__ overlays/ # Pasta que o argo irá monitorar
                          |__ env-qa.yaml  # Se necessário modificar variáveis de ambiente.
                          |__ kustomization.yaml
    
          OBS: Arquivo kustomization.yaml será explicado mais abaixo.
   ```
   
  ## Explicando os arquivos de configuração "Type: Application" 
    
  > O tipo de arquivo Application é responsável pelas configurações de monitoramento do gitlab para o ArgoCD, nele você consegue setar:
    
  >    - URL do repositório que você quer monitorar
  >    - Branch e caminho do repositório para monitorar.
  >    - Endpoint do cluster destino, se você trabalhar com multi-clusters.
  >    - Namespace que será criado pelo argocd no cluster.
    
  >   Políticas de sincronização:
  >    - prune: Quando setado "true" deleta os serviços que não estão sincronizados com os arquivos do gitlab.
  >    - selfHeal: Quando setado "true" proíbe quaisquer modificações diretas nos arquivos de deployments que não sejam a partir do gitlab.
  >    - allowEmpty: Quando setado "true" permite monitorar repositórios gitlabs que não tenham nenhum arquivo ainda commitado.
    
  > Opções de sincronização:
  >    - CreateNamespace=true , permite a criação de namespace pelo argocd, caso não existam no cluster destino.
  >    - PruneLast=true, exclue a réplica anterior.
  >    - PrunePropagationPolicy, políticas de exclusão do kubernetes, mais informações aqui: https://kubernetes.io/docs/concepts/architecture/garbage-collection/#controlling-how-the-garbage-collector-deletes-dependents.
  
  #### Adding settings
    
  ```
    ## Exemplo de arquivo application

    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
        name: application-teste
        namespace: argocd
        finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
        project: teste
        source:
            repoURL: < Url do Projeto onde encontra-se a pasta Overlays >
            targetRevision: < Branch do projeto >
            path: < Diretório dentro do projeto que você quer monitorar >
        destination:
            server: < Endpoint do cluster kubernetes >
            namespace: teste
        syncPolicy:
            automated:
                prune: true
                selfHeal: true
                allowEmpty: true
            syncOptions:
            - PrunePropagationPolicy=foreground
            - CreateNamespace=true
            - PruneLast=true
            retry:
                limit: 5
                backoff:
                    duration: 5s
                    factor: 2
                    maxDuration: 1m
  ```
   
  ## Explicando os arquivos de configuração "Type: Project" 
   
  > O arquivo tipo Project é o responsável por setar regras para os arquivos applications, como permissões de criação de determinados serviços dentro de um cluster (configmaps, deployments), permissões de deploy em cluster específico, lista de repositórios gitlab que são permitidos monitorar, entre outros recursos.
  
  ```
    ## Exemplo de arquivo type:Project
    
    apiVersion: argoproj.io/v1alpha1
    kind: AppProject
    metadata:
        name: teste
        namespace: argocd
    spec:
      description: Arquivos ambiente de teste
      sourceRepos:
      - "*"       # Applications vinculadas a este projeto podem monitorar qualquer repositório no gitlab.
      destinations:
      - namespace: 'teste*'   # Appications vinculadas a este projeto podem ser deployadas em qualquer namespace que comece com teste.
        server: < Endpoint do cluster kubernetes >

      # Só é permitido o argo criar namespaces dentro do cluster.
      clusterResourceWhitelist:
      - group: ''
        kind: 'Namespace'

      # Só é permitido ao argo deployar deployments, services and ingresses dentro do namespace.
      namespaceResourceWhitelist:
      - group: 'apps'
        kind: Deployment
      - group: ''
        kind: Service
      - group: 'networking.k8s.io'
        kind: Ingress
  ```
   ## Explicando Kustomize
    
   > A ferramenta kustomize, resumidamente, é útil na criação de templates para separação dos arquivos .yaml em partes e no fim dar um merge em todos os templates em um só.
    
   > O argocd tem compatibilidade com Helm e Kustomize sem precisar de nenhuma instalação de nossa parte.
   >   - Com o Helm, o ArgoCD procura um arquivo nomeado Chart.yaml e se existir ele usa as configurações de deployment do helm.
   >   - Pro caso do Kustomize, o ArgoCD procura um arquivo nomeado kustomization.yaml e se existir, utiliza as configurações de deployment do kustomize.
    
   > Particularmente, acho o Kustomize uma ferramenta um tanto mais poderosa e de mais fácil entendimento pro cenário do argocd.
   
   ### Organização dos arquivos nas pastas
   
   > Como apresentado a estrutura lá em cima, terão dois arquivos kustomization.yaml na esturutra de pastas.
    
   > O primeiro kustomization.yaml servirá para coletar os resources da pasta bases, que será utilizado pelo segundo kustomization.yaml na pasta overlays.
   
   ```
    # kustomization.yaml na pasta bases/
    # Na tag "resources" é especificado os arquivos da aplicação que serão deployados (necessários está no mesmo diretório do arquivo kustomization.yaml)
    
    resources:
      - application-deployment.yaml
      - application-clusterservice.yaml
  ```
  
  > O segundo kustomization.yaml servirá, de fato, para enviar os arquivos para deploy no cluster.
   
  ```
    # kustomization.yaml na pasta overlays/
    
    bases:
      - ../bases     # Caminho para o diretório bases/

    images:
    - name: <NomeDaImagem>
      newName: <NomeDaImagem>                   # Aqui é possível modificar imagens baseado em mundaças na aplicação, basta mudar a versão da "newTag"
      newTag: <TagNovaQueDeseja>

    patchesStrategicMerge:
      - arquivo.yaml                            # Caso precise modificar alguma coisa nos arquivos na pasta bases/ é possível solicitar um merge com as mudanças.

  ```
    
  ## Leituras complementares
    
  >    - https://argoproj.github.io/argo-cd/user-guide/kustomize/
  >    - https://argoproj.github.io/argo-cd/user-guide/projects/
  >    - https://argoproj.github.io/argo-cd/user-guide/private-repositories/
  >    - https://argoproj.github.io/argo-cd/user-guide/auto_sync/
  
  >    - https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/
  >    - https://speakerdeck.com/spesnova/introduction-to-kustomize
    
  </body>
</html>
