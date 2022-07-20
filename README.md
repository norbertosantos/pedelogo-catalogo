# DESAFIO DEVOPS PRO NAMESPACE

Esse exercício faz parte do desafio da aula de namespace do curso Devops Pro.

Neste exercício, precisamos comunicar a nossa Api do Pedelogo que está dentro de um namespace chamado pedelogo-api com a sua base de dados em Mongo Db que encontra-se em um namespace chamado mongodb.

## 1- Criando os namespaces:

Para este exercício, vamos criar dois namespaces. O primeiro namespace chamado pedelogo-api que vai conter a api do pedelogo e o segundo namespace mongodb que vai conter o nosso banco de dados em MongoDb. Para este fim usaremos o seguinte comando:

```docker
kubectl create namespace pedelogo-api
kubectl create namespace mongodb-pedelogo 
````

## 2- Criando os descritores da api do pedelogo:

Para organizar o nosso projeto, dentro da pasta k8s, vamos criar dois objetos Kubernetes. Um Deployment que chamaremos deployment.yaml e um service do tipo NodePort chamado service.yaml.

**deployment.yaml**

```docker
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  namespace: pedelogo-api
spec:
 selector:
   matchLabels:
     app: api
 template:
  metadata:
   labels:
     app: api
  spec:
    containers:
      - name: api
        image: norbertosantos/pedelogo-catalogo:v1.0.0 
        ports:
          - containerPort: 80
            name: http 
          - containerPort: 443
            name: https 
        imagePullPolicy: Always
        env:
            - name: Mongo__Host
              value: "service-pedelogo-mongodb"
            - name: Mongo__User
              value: "mongouser"
            - name: Mongo__Password
              value: "mongopwd"
            - name: Mongo__Port
              value: "27017"
            - name: Mongo__Database
              value: "admin" 
```

**service.yaml**

```docker
apiVersion: v1 
kind: Service 
metadata: 
  name: api-service 
  namespace: pedelogo-api
spec:
  selector:
    app: api 
  ports:
    - port: 80
      targetPort: 80
      name: http 
    - port: 443
      targetPort: 443
      name: https
  type: NodePort
````

Agora, vamos criar os nossos objetos para api do pedelogo no nosso cluster, utilizando os seguintes comandos: 

```docker
kubectl apply -f deployment.yaml -n pedelogo-api
kubectl apply -f service.yaml -n pedelogo-api
````

## 3- Criando os descritores do mongodb:

Para organizar nosso projeto, dentro da pasta k8s, vamos criar uma pasta chamada mongodb para armazenar dois objetos Kubernetes para o Mongo Db. Um Deployment e um Service.

**deployment.yaml**

```docker
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  namespace: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb 
    spec:
      containers:
        - name: mongodb
          image: mongo:4.2.8
          ports:
           - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              value: mongouser
            - name: MONGO_INITDB_ROOT_PASSWORD
              value: mongopwd
```
**service.yaml**

```docker
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  namespace: mongodb
spec:
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
  type: ClusterIP

```
Vamos criar os nossos objetos Kubernetes dentro do namespace mongodb utilizando os seguintes comandos:

```docker
kubectl apply -f deployment.yaml -n mongodb
kubectl apply -f service.yaml -n mongodb
```

## 4- Criando os External Services:

External Services é um tipo de serviço dentre os objetos de serviço do Kubernetes que nos permite criar uma referência externa a um serviço localizado em um determinado namespace.

Por padrão, ao criamos um serviço em um namespace, o Kubernetes cria um DNS reapeitando a seguinte estrutura:

**nome-service.nome-namespace.svc.cluster.local**

Para o nosso exercício, vamos criar dois External Services:

Um External Services para ligar a api do pedelogo localizada no namespace pedelogo-api com o seu banco de dados mongodb no namespace mongodb. Este External Service chamaremos de service-mongodb.

Um segundo External Services para ligar o namespace default do nosso cluster com a api do pedelogo que está localizada no namespace pedelogo-api que chamaremos de service-pedelogo-api.

**service-mongodb.yaml**

```docker
apiVersion: v1 
kind: Service
metadata:
  name: service-pedelogo-mongodb
  namespace: pedelogo-api
spec:
  type: ExternalName
  externalName: mongodb-service.mongodb.svc.cluster.local
````

**service-pedelogo-api.yaml**

```docker
apiVersion: v1
kind: Service 
metadata:
  name: service-pedelogo-api
spec:
  type: ExternalName
  externalName: api-service.pedelogo-api.svc.cluster.local  
```

Após criarmos o nosso External Services **service-mongodb.yaml**, nós vamos precisar referenciar o mesmo nas variáveis de ambiente do nosso objeto de Deployment da nossa api do pedelogo.Abaixo, segue o trecho do nosso Deployment da api do Pedelogo onde faremos a referência.

**deployment.yaml Api Pedelogo**

```docker
spec:
    containers:
      - name: api
        image: norbertosantos/pedelogo-catalogo:v1.0.0 
        ports:
          - containerPort: 80
            name: http 
          - containerPort: 443
            name: https 
        imagePullPolicy: Always
        env:
            - name: Mongo__Host
              value: "service-pedelogo-mongodb"
            - name: Mongo__User
              value: "mongouser"
            - name: Mongo__Password
              value: "mongopwd"
            - name: Mongo__Port
              value: "27017"
            - name: Mongo__Database
              value: "admin"

```

Uma vez criado os nossos External Services, conforme descrito anteriormente. Vamos criar os nossos External Services, estando atentos ao namespace onde eles precisam ser criados. O service-mongodb criamos no namespace **pedelogo-api** . O service-pedelogo-api criamos no namespace **default** .
Para criarmos nossos External Services, usaremos os seguintes comandos:

```docker
kubectl apply -f service-mongodb.yaml -n pedelogo-api
kubectl apply -f service-pedelogo-api.yaml
```
## 5- Entendendo os objetos Kubernetes criados.

Depois de realizarmos os passos anteriores, nossos objetos Kubernetes serão os seguintes:

No namespace **pedelogo-api**, teremos:

**Um Deployment** da Api do Pedelogo.
**Um Service** da Api do Pedelogo.
**Um ReplicaSet** da Api do Pedelogo.
**Um Pod** da Api do Pedelogo.
**Um External Service** para ligar a api do pedelogo ao banco mongodb. 

Esses objetos podem ser verificados utilizando o seguinte comando:

```docker
kubectl get all -n pedelogo-api
```
No namespace **mongodb**, teremos:

**Um Deployment** do Mongodb.
**Um Service** do MongoDb.
**Um ReplicaSet** do MongoDb.
**Um Pod** do MongoDb.

Esses objetos podem ser verificados utilizando o seguinte comando:

```docker
kubectl get all -n mongodb
````
No namespace default, teremos: 

Um External Service para permitir a comunicação entre o namespace default do nosso cluster e o namespace pedelogo-api onde está localizado o service da Api do Pedelogo.

Esses objetos podem ser verificados utilizando o seguine comando:

```docker
kubctl get all
```
## 6- Testando a nossa aplicação:

Para testar a nossa aplicação do Pedelogo. Eu criei dois produtos utilizando uma ferramenta para execução de testes em APIS REST chamada Postman.

Antes de executar as chamadas no Postman, vamos criar um port-forward para nosso serviço da api Pedelogo para que possamos acessar através do navegador. Vamos utilizar o seguinte comando:

```docker
kubectl port-forward -n pedelogo-api service/api-service 8080:80
```

[Postman]("https://www.postman.com/")

**POST http://localhost:8080/produto** 

JSON:
 ```json
{
    "nome":"I2GO 20000Mh",
    "preco":200,
    "categoria":"portateis"
}
```

POST http://localhost:8080/produto 

JSON:

```json 
{
    "nome":"Iphone 13 Pro",
    "preco":8250,
    "categoria":"celulares"
}
```
Para testarmos a nossa aplicação para verificar se ela nos retorna a lista de produtos que cadastramos anteriormente, vamos digitar no nosso navegador, o seguinte endereço:
**http://localhost:8080/produto**

Como JSON de resposta a nossa chamada, receberemos:

```json
[{"id":"62d715d663156b819fc6e608","nome":"I2Go 20000","preco":200,"categoria":"portateis"},{"id":"62d7161863156b819fc6e609","nome":"Iphone 13 Pro","preco":8250,"categoria":"celulares"}]
```

Para maiores informações sobre o uso de namespaces nas nossas aplicações no Kubernetes. Acesse:
[Documentação Oficial Namespaces Kubernetes]("https://kubernetes.io/pt-br/docs/concepts/overview/working-with-objects/namespaces/")
