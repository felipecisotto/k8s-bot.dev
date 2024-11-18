## O que é?

É um orquestrado e gerenciador de containers, que geralmente são criados por docker, e que tem como responsabilidade, scalar, distribuir e lidar com a conexão desses containers

## Pods

São os containers, ou conjunto de containers. 
> "Pods are the smallest deployable units of computing that you can create and manage in Kubernetes."

Os pods devem ser efemeros, ou seja, devem ser temporários, eles podem apresentar falhas e serem substituidos sem nenhum problema e a qualquer momento. 
isso ajuda a promover a alta dispobinilidade e a imutabilidade de pod, pois se ele pode ser recriado, ele deve ser recriado igual foi a primeira vez. 

Portanto não devemos armazenar dados nos containers

Cada pod vai ter o seu único endereço de ip, podemos ver isso usando o comando 
```bash 
kubectl get pods -o wide
```

teremos o output:
```bash
NAME                              READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
synergychat-web-fbdb8d849-2xv72   1/1     Running   0          69s     10.244.0.7   minikube   <none>           <none>
synergychat-web-fbdb8d849-sq66s   1/1     Running   0          6m23s   10.244.0.6   minikube   <none>           <none>

```

#### Thrashing Pods

São containers que estão travados durante o start e ficam sempre reiniciando

## Nodes
Os nodes são os servidores que rodam o k8s 

## Deploy docker image

```bash 
kubectl create deployment synergychat-web --image=bootdotdev/synergychat-web:latest
```

o comando utilizado para deployar uma imagem é o create comand, nesse caso, passamos uma imagem docker, no exemplo acima pegamos uma imagem do docker hub

### Port Foward

Uma vez que o serviço está rodando não é possivel acessar a uma porta, então é necessário adicionar o port foward para ele 
```bash 
kubectl port-forward PODNAME 8080:8080
```

## Deployment 

é a declaração do serviço que será deployado , aqui temos um exemplo de yaml

```yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergychat-api
  namespace: default
  labels:
    app: synergychat-api
spec:
  selector:
    matchLabels:
      app: synergychat-api
  replicas: 1
  template:
    metadata:
      labels:
        app: synergychat-api
    spec:
      containers:
      - name: synergychat-api
        image: bootdotdev/synergychat-api:latest
		env:
		  - name: API_PORT
			valueFrom:
			  configMapKeyRef:
				name: synergychat-api-configmap
				key: API_PORT
	```

apiVersion -> indica o tipo do serviço e que será um app no formato v1
kind -> indica o tipo do arquivo 
metadata.name -> indica o nome do deployment
metadata.namespace -> em qual namespace vai estar o container
labels.app -> label do app 
spec.selector.matchLabels.app -> 
spec.replicas -> quantidade de replicas 
spec.containers -> lista com os contaneirs que vão fazer parte do pod, no caso tempo nome e imagem
env: para setar as configs do config map
## Config Maps 

**Essas configs não podem ser usados para dados sensiveis.**

config Maps é uma forma de desacoplar as configurações dos containers, então é adicionado um novo arquivo, com todas as variaveis de ambiente necessária e não será necessário rebuildar a imagem caso seja necessário trocar alguma config


exemplo de arquivo 
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: synergychat-api-configmap
  namespace: default
data:
  API_PORT: "8080"
```
dentro do data, adicionamos todos os nossos dados e chaves

## Services 
São endpoints estaveis que sempre vão estar disponiveis e direcionando o trafego para os pods que estão ativos
![[Pasted image 20241113225531.png]]

Arquivo de exemplo:

```yaml 
# https://kubernetes.io/docs/concepts/services-networking/service/
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: default
spec:
  selector:
    app: synergychat-web
  type: ClusterIP
  ports:
  - name: web-service
    protocol: TCP
    port: 80
    targetPort: 8080
---
```

#### Tipos de serviço 

- ClusterIP: é um IP dentro do cluster e interno, deve ser usado para expor os pods dentro do cluster 
- NodePort: Expõe o serviço em um IP em cada node do k8s
- LoadBalancer: Cria um load balancer externo na infra-estrutura de cloud (se suportado)
- ExternalName: Mapea o serviço para um nome de DNS
**Qual devo usar?**
Quando queremos usar o serviço apenas dentro do cluster ClusterIp e quando vamos expor o serviço para o moundo LoadBalancer
Provavelmente em aplicações do mundo real, vamos usar o **Ingress** para expor a app ao mundo 

## Ingress

O recurso de ingress é usado para expor os serviços para o mundo externo e é usado frequentemente em produção: 

>An Ingress may be configured to give Services externally-reachable URLs, load balance traffic, terminate SSL / TLS, and offer name-based virtual hosting. An Ingress controller is responsible for fulfilling the Ingress, usually with a load balancer, though it may also configure your edge router or additional frontends to help handle the traffic.
>
>An Ingress does not expose arbitrary ports or protocols. Exposing services other than HTTP and HTTPS to the internet typically uses a service of type Service.Type=NodePort or Service.Type=LoadBalancer.

![[Pasted image 20241113232411.png]]

**Cada cloud provider pode ter certas configurações para o Ingress**

## Storage 

#### Volumes Efemeros

Podemos adicionar para os nossos containers, para que eles usem um volume dentro do pod, Esses containers não serão persistidos caso o pod seja deletado, porém, pode ser útil para compartilhar pastas entre containers dentro do mesmo pod.
por exemplo: 

1. Adicionando `volumes` para a secção do `spec/template/spec`.

```yaml
volumes:
  - name: cache-volume
    emptyDir: {}
```

2. adicionando uma nova secção de  `volumeMounts` aos containers. Isso irá montar o volume dentro do container .

```yaml
volumeMounts:
  - name: cache-volume
    mountPath: /cache
```

#### Persistente Volumes

Podemos crias 2 tipos de volumes persistentes: 
1. PV estáticos, que são criados e gerenciados manualmente pelo admin do cluster 
2. PV dinamicos, são criados automaticamente quando um pod precisa utilizar o PV

#### Persistent Volume Claims (PVC)

Um PVC é uma requisição para criar um vpc, isso solicita a criação do pv e deixa ele pronto para ser utilizado pelo pod.
```yaml
# https://kubernetes.io/docs/concepts/storage/persistent-volumes/
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: synergychat-api-pvc
  labels:
    app: synergychat-api-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

- `metadata/name`: nome do vpc
- `spec/accessModes`:  temos as seguintes opções 
	- `ReadWriteOnce`: Pode ser acessado por apenas 1 node, mas pode vários pods dentro desse node 
	- `ReadOnlyMany`: Pode ser lido por todos os nodes
	- `ReadWriteMany`: Pode ser lido e escrito por todos os nodes
	- `ReadWriteOncePod`: Só pode ser lido e escrito por 1 pod em 1 node
- `spec/resources/requests/storage`: `1Gi`

#### Utilizando um PV 

Para utilizar um pv em um pod é necessário criar um volume no pod e dizer que esse volume pertence a um PVC, o processo é o mesmo que foi utilizado nos [[#Volumes Efemeros]]

**Declarando o volume**

```yaml
volumes:
  - name: synergychat-api-volume
    persistentVolumeClaim:
      claimName: synergychat-api-pvc
```

**Montando o volume no pod**

```yaml
volumeMounts:
  - name: synergychat-api-volume
    mountPath: /persist
```

## Namespaces

Namespaces é uma forma de você isolar um determinado grupo de recursos, é como se fossem diretorios do pc e isso serve muito bem quando temos times diferentes trabalhando em aplicação diferentes e não queremos que 1 influencie na outra. Podemos ter os mesmos recursos em diferente namespaces.

Para utilizar ou determinar o namespace que queremos mexer, podemos usar o `-n` ou `--namespace`  no comando do `kubectl`, se não fizermos isso, vamos utilizar o namespace `default` 

**Para criar uma nova namespace, podemos usar o comando**

```bash 
kubectl create ns crawler
```

Para determinar em qual namespace deve ficar cada item, temos que adicionar a tag namespace dentro do metadata do arquivo que estamos modificando, isso vale para todos os tipos de objeto que estamos criando 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergychat-api
  namespace: default
  labels:
    app: synergychat-api
```

#### DNS 

Para acessar os recursos em outros namespaces, podemos usar o dns que o próprio k8s cria com  `http://<service-name>.<namespace>.svc.cluster.local`, onde  o  `svc.cluster.local` pode ser omitido e se os objetos estão na mesma namespace, também podemos omitir o namespace. 

## Resources 

Para determinar os recursos que os pods podem utilizar, temos 2 coisas que podemos fazer, definir o `requests` e o `limit`

- Requests: É o que vamos solicitar ao k8s quando o pod iniciar, uma vez definido isso, o pod irá iniciar com essa quantidade de recurso que foi definida, caso a soma dos requests do pods ultrapasse os recursos da maquina, o pod vai se rejeitar a subir 
- limit: o Limit é utilizado para proteção do cluster, e uma vez esses valores definidos, os pods não poderão passar desse uso, não devemos utilizar sempre os limites, pois isso pode gerar problemas para as aplicações.

Para o criador curso a regra de ouro é: 

> - Set memory _requests_ ~10% higher than the average memory usage of your pods
> - Set CPU _requests_ to 50% of the average CPU usage of your pods
> - Set memory _limits_ ~100% higher than the average memory usage of your pods
> - Set CPU _limits_ ~100% higher than the average CPU usage of your pods

Para definir esses valores, devemos adicionar a cada container do nosso pod os valores: 

```yaml
spec:
  containers:
  - name: <container-name>
    image: <image-name>
    resources:
      limits:
        memory: <max-memory>
        cpu: <max-cpu>
	  requests:
	    memory: 40000Mi
	    cpu: 10m
```

#### Unidades de medida 

Para definir a quantidade de memória podemos usar
 - Ki -> Kilobytes
 - Mi -> Megabytes
 - Gi -> Gigabytes
 Para definir a quantidade de cpu utilizamos a mili-cores (m), por exemplo 500m é 0,5 cores de cpu

## # Horizontal Pod Autoscaling (HPA)

Para determinarmos como nossos pods devem escalar, temos que criar um novo objeto de HPA, indicando para o sistema devemos escalar a nossa app, para isso funcionar não podemos ter determinado a quantidade de replicas no deployment

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: testcpu-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <deployment-name>
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

nesse exemplo estamos criando o `testcpu-hpa` que tem no minimo 1 replica e no maximo 10 e escala quando a cpu do pod chega 50%
