# Amdocs TechInnovation Week - 2024

Este repositório contém a os códigos e documentação utilizados no Webinar apresentado pelo time de DevOps no TechInnovation Week 2024 da Amdocs.
O intuito é demonstrar a utilização de Inteligência Artificial aplicada no escopo de Devops, mais especificamente em ambientes Kubernetes.  

# 1. Preparação do ambiente

Para este Webinar, utilizaremos o K3D para criar um cluster kubernetes, Argo CD para gerenciar os deployments e o K8sGPT para monitorar e recomendar soluções das nossas aplicações.  

## 1.1 Criação e configuração do cluster

Vamos criar o cluster **techinnovation** que escutará na porta 6443.

```
k3d  cluster  create  techinnovation  --api-port  6443  -p  "8080:80@loadbalancer"  -p  "8443:443@loadbalancer"  -p  "8090:8090@loadbalancer"  --k3s-arg  "--tls-san=192.168.68.61"@server:*  --kubeconfig-update-default  --timestamps
```

Salve o seu arquivo de **KUBECONFIG** que contém as informações necessárias para acessar o seu cluster:

```
k3d  kubeconfig  get  techinnovation  >  techinnovation.config
export  KUBECONFIG=./techinnovation.config
```

Agora vamos expor as portas **30000** e **30001** que serão utilizadas pelo Argo CD.

```
k3d  cluster  edit  techinnovation  --port-add  30000:30000@loadbalancer
k3d  cluster  edit  techinnovation  --port-add  30001:30001@loadbalancer
```
## Deploy do Argo CD

Para fazer o deploy do Argo CD, utilizaremos o HELM, que é um gerenciador de pacotes para kubernetes.

```
helm  install  argocd  oci://registry-1.docker.io/bitnamicharts/argo-cd  -n  argocd  --create-namespace
```

Para que o Argo CD seja acessível fora do cluster, precisamos expor o service utilizando NodePort ou então criando um Ingress. Para este exemplo utilizaremos NodePort com as portas expostas anteriormente (portas 30000 e 30001).

```
kubectl edit svc argocd-argo-cd-server -n argocd
```

Altere o type de **ClusterIP** para **NodePort** e adicione as portas 30000 para HTTP e 30001 para HTTPS.

    apiVersion: v1
    kind: Service
    metadata:
      annotations:
        meta.helm.sh/release-name: argocd
        meta.helm.sh/release-namespace: argocd
      labels:
        app.kubernetes.io/component: server
        app.kubernetes.io/instance: argocd
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: argo-cd
        app.kubernetes.io/version: 2.12.3
        helm.sh/chart: argo-cd-7.0.13
      name: argocd-argo-cd-server
      namespace: argocd
    spec:
      clusterIP: 10.43.62.40
      clusterIPs:
      - 10.43.62.40
      externalTrafficPolicy: Cluster
      internalTrafficPolicy: Cluster
      ipFamilies:
      - IPv4
      ipFamilyPolicy: SingleStack
      ports:
      - name: http
        nodePort: 30000		<--- Adicione esta linha para HTTP
        port: 80
        protocol: TCP
        targetPort: http
      - name: https
        nodePort: 30001		<--- Adicione esta linha para HTTPS
        port: 443
        protocol: TCP
        targetPort: http
      selector:
        app.kubernetes.io/component: server
        app.kubernetes.io/instance: argocd
        app.kubernetes.io/name: argo-cd
      sessionAffinity: None
      type: NodePort		<--- Altere de ClusterIP para NodePort

Para testar, acesse o Argo CD com a URL abaixo substituindo o IP pelo endereço do seu host.

    https://<IP>:30001/

Para fazer o login, decodifique o password do usuário **admin**:

```
kubectl -n argocd get secret argocd-secret -o jsonpath="{.data.clearPassword}" | base64 -d
```

## Instalação do K8sGPT Operator

Novamente, utilizaremos o HELM para fazer o deploy do K8sGPT Operator.

```
helm repo add k8sgpt https://charts.k8sgpt.ai/  
helm repo update
helm  install  k8sgpt  k8sgpt/k8sgpt-operator  -n  k8sgpt-operator-system  --create-namespace
```

Para aumentar a capacidade do K8sGPT, podemos integrado com alguma IA Generativa, como OpenAI, Ollama e outras. Para este exemplo, utilizaremos o Ollama que é uma Local AI.

```
kubectl -n k8sgpt-operator-system apply  -f  k8sgpt/k8sgpt-ollama.yaml
```

Altere o YAML acima com o endereço IP/Porta do host onde o Ollama está sendo executado.

    apiVersion: core.k8sgpt.ai/v1alpha1
    kind: K8sGPT
    metadata:
      name: k8sgpt-ollama
    spec:
      ai:
        enabled: true
        model: llama3.1
        backend: localai
        baseUrl: http://<IP>:11434/v1
      noCache: false
      version: v0.3.41

Também iremos habilitar a integração do K8sGPT com o Trivy, um scanner de vulnerabilidades. 

```
k8sgpt integration activate trivy
k8sgpt integration list
```

Pronto, seu ambiente está configurado! =)

# 2. Fazendo o Deploy de Aplicações com o Argo CD

Para este Webinar, faremos o deploy de duas aplicações via backend criando manifests no kubernetes. Porém o mesmo pode ser feito utilizando a interface gráfica do Argo CD.


### App1

Um container NGINX que contém uma imagem inexistente.
Neste exemplo utilizaremos a opção de sync automático do Argo CD.

```
kubectl -n argocd apply -f argocd-applications/app1.yaml
```
### App2

Um container BUSYBOX que contém um erro na quantidade de CPUs requerida.
Para este exemplo, utilizaremos a opção de sync manual do Argo CD.

```
kubectl -n argocd apply -f argocd-applications/app2.yaml
```

# Utilizando o K8SGPT para monitorar nossas aplicações

O K8sGPT tem a capacidade de monitorar os recursos do kubernetes e identificar potenciais problemas e recomendar soluções.

Quando utilizado o K8SGPT operator, este operador cria objetos **Result** para descrever os problemas encontrados e soluções.

```
kubectl -n k8sgpt-operator-system get result
kubectl -n k8sgpt-operator-system describe result
```

Também pode-se utilizar o client do K8sGPT para analizar seus manifests e pedir explicação sobre o problema. Para isto, o K8sGPT utilizará a integração com uma IA Generativa.

```
k8sgpt  analyze
```

ou se preferir uma explicação mais detalhada sobre o problema:

```
k8sgpt  analyze  --explain
```

## Monitorar determinados recursos e/ou namespaces

Para monitorar um determinado tipo de recurso e/ou restringir o escopo para uma namespace específica, pode-se utilizar os parâmetros **-f** para filtrar por um determinado tipo e **-n** para definir a namespace.

```
k8sgpt analyze --explain -n app1 -f Pod
```  

## Definir idioma

Também é possível definir o idioma de sua preferência utilizando o parametro **-l**

```
k8sgpt analyze --explain -n app1 -f Pod -l Portuguese
```

## Vulnerabilidades

Também é possível monitorar vulnerabilidades (CVEs) identificadas pelo Trivy.

```
k8sgpt  analyze  -f  VulnerabilityReport  --explain
```


# Documentação

### Kubernetes
Documentação official do kubernetes: [https://kubernetes.io/docs/home/]

## Argo CD
Documentação official do Argo CD: [https://argo-cd.readthedocs.io/en/stable/]

## K8sGPT
Documentação official do K8sGPT: [https://docs.k8sgpt.ai/]



