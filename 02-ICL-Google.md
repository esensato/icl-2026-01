## Google Cloud
- Acesso ao console web https://console.cloud.google.com/
#### gcloud
- Exibir a versão
```bash
gcloud -v
```
- Listando as configurações
```bash
gcloud config list
```
- Obtendo o valor de uma configuração dentro da sessão `[core]` (para definir um valor, utilizar o `set`)
```bash
gcloud config get account
```
#### Projetos
- Listar todos os projetos criados
```bash
gcloud projects list

gcloud projects list --filter="name=('meu-projeto-123')"
```
- Criar um projeto e definí-lo no contexto (projeto de trabalho atual)
```bash
gcloud projects create app-project-$USER

gcloud config set project app-project-$USER
```
- Obter informações do projeto
```bash
gcloud projects describe $(gcloud config get-value project)
```
- Listar todos os recursos associados a um projeto
```bash
gcloud services list --enabled
gcloud asset search-all-resources --scope=projects/app-project-$USER --format="table(assetType,name,location)"
```
#### Acessando Via RESTFul
- Obter o *token* de acesso
```bash
gcloud auth print-access-token
```
- Realizar a requisição
```bash
curl -H "Authorization: Bearer $(gcloud auth print-access-token)" "https://cloudresourcemanager.googleapis.com/v1/projects"
```
#### Acessando Via Nodejs
- Criar a estrutura do projeto
```bash
mkdir gcp-projects-demo
cd gcp-projects-demo
npm init -y
```
- Instalar as dependências
```bash
npm install --save google-auth-library axios
```
- Código exemplo (para listar os projetos)
```bash
const { GoogleAuth } = require('google-auth-library');
const axios = require('axios');

async function listarProjetos() {
    try {
        // Usa credenciais já autenticadas (Cloud Shell, gcloud auth, service account etc.)
        const auth = new GoogleAuth({
            scopes: ['https://www.googleapis.com/auth/cloud-platform.read-only']
        });

        const client = await auth.getClient();
        const token = await client.getAccessToken();

        const response = await axios.get(
            'https://cloudresourcemanager.googleapis.com/v1/projects',
            {
                headers: {
                    Authorization: `Bearer ${token.token}`
                }
            }
        );

        console.log("Projetos encontrados:\n");

        response.data.projects.forEach(project => {
            console.log(`- ${project.projectId} (${project.name})`);
        });

    } catch (erro) {
        console.error("Erro:", erro.response?.data || erro.message);
    }
}

listarProjetos();
```
#### Contas de Faturamento
- Listar contas de faturamento
```bash
gcloud beta billing accounts list --filter="open=true"
```
- Vincular o projeto a uma conta de cobrança (obtida acima)
```bash
gcloud beta billing projects link app-project-$USER --billing-account=<OBTIDA_ACIMA>
```
- Remover um projeto e todos os seus componentes
```bash
gcloud projects delete <NOME_DO_PROJETO>
```
#### APIs
- Para habilitar uma determinada API no projeto atual
```bash
gcloud services enable compute.googleapis.com
```
- Listas as APIs habilitadas para o projeto atual
```bash
gcloud services list --enabled | grep compute
```
- Desabilitar uma API
```bash
gcloud services disable compute.googleapis.com
```
#### Regions e Zones
- Regiões (*locations*) Google Cloud [https://cloud.google.com/about/locations](https://cloud.google.com/about/locations)
- Listar *regions* e *zones*
```bash
gcloud compute regions list
gcloud compute zones list
gcloud compute firewall-rules list --filter="NETWORK:'default' AND ALLOW:'icmp'"
```
- Para definir uma *zone*
```bash
gcloud config set compute/region REGION
```
#### Vairáveis de Ambiente
- Úteis para referenciar valores a serem utilizados em *scripts*
```bash
export PROJECT_ID=$(gcloud config get-value project)
export ZONE=$(gcloud config get-value compute/zone)

echo $PROJECT_ID
echo $ZONE
```
#### Máquinas Virtuais
```bash
gcloud compute machine-types list --zones southamerica-west1-a

gcloud compute machine-types list --zones southamerica-west1-a --filter="name=('e2-micro')"

gcloud compute machine-types list --zones southamerica-west1-a --filter="name=('e2-medium')"

gcloud compute instances create vm-1 --machine-type e2-micro --zone southamerica-west1-a

gcloud compute instances list
```
- Além da VN também é criado um disco de armazenamento
```bash
gcloud compute disks list
```
- Efetuando uma conexão com a VM criada
```bash
gcloud compute ssh vm-1 --zone southamerica-west1-a
```
- Instalando um servidor *http* dentro da VM
```bash
sudo apt update && sudo apt install -y nginx

echo "VM $(hostname)" | sudo tee /var/www/html/index.html

exit
```
- Identificar a VM por meio de uma *tag*
```bash
gcloud compute instances add-tags vm-1 --tags http-server,https-server --zone southamerica-west1-a
```
- Listando regras de *firewall*
```bash
gcloud compute firewall-rules list --filter="NETWORK:'default' AND ALLOW:'icmp'"
```
- Adicionar regra ao *firewall*
```bash
gcloud compute firewall-rules create default-allow-http --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server

gcloud compute firewall-rules list --filter=ALLOW:'80'
```
- Testar utilizando o `curl`
```bash
curl http://$(gcloud compute instances list --filter=name:vm-1 --format='value(EXTERNAL_IP)')
```
#### Balanceamento Carga
- Criar uma segunda VM (chamada **vm-2**) definindo uma *tag* para ela
```bash
gcloud compute instances add-tags vm-2 --tags http-server,https-server --zone southamerica-west1-a
```
- Criar um grupo de instâncias e adicionar as VMs
```bash
gcloud compute instance-groups unmanaged create nginx-group --zone=southamerica-west1-a
gcloud compute instance-groups unmanaged add-instances nginx-group --zone=southamerica-west1-a --instances=vm-1,vm-2

gcloud compute instance-groups list
```
- Criar uma porta em comum para acesso ao grupo de instâncias das VMs
```bash
gcloud compute instance-groups set-named-ports nginx-group --named-ports=http:80 --zone=southamerica-west1-a
```
- Criar o mecanismo de *health check* para determinar de as MVs estão de fato executando o serviço na porta 80 (padrão *http*)
```bash
gcloud compute health-checks create http nginx-health-check --port=80
```
- Criar o serviço de *backend* e adicionar o grupo de VMs
```bash
gcloud compute backend-services create nginx-backend --protocol=HTTP --health-checks=nginx-health-check --global
gcloud compute backend-services add-backend nginx-backend --instance-group=nginx-group --instance-group-zone=southamerica-west1-a --global
```
- Criar o mapeamento de URL
```bash
gcloud compute url-maps create nginx-map --default-service=nginx-backend
```
- Instanciar o servidor *proxy* que irá realizar o balanceamento
```bash
gcloud compute target-http-proxies create nginx-proxy --url-map=nginx-map
```
- Adicionar a regra de *firewall*
```bash
gcloud compute forwarding-rules create nginx-rule --global --target-http-proxy=nginx-proxy --ports=80
```
- Obter o endereço de IP para acesso
```bash
gcloud compute forwarding-rules list
```
- Verificar a saúde das VMs
```bash
gcloud compute backend-services get-health nginx-backend --global
```
- Para parar e remover a instância da VM
```bash
gcloud compute instances stop vm-1 --zone=southamerica-west1-a
gcloud compute instances delete vm-1 --zone=southamerica-west1-a
```
- Para remover o que foi criado
```bash
gcloud compute firewall-rules delete default-allow-http
gcloud compute forwarding-rules delete nginx-rule --global
gcloud compute target-http-proxies delete nginx-proxy
gcloud compute url-maps delete nginx-map
gcloud compute backend-services delete nginx-backend --global
gcloud compute health-checks delete nginx-health-check
gcloud compute instance-groups unmanaged delete nginx-group --zone=southamerica-west1-a
```
## Google Cloud Run
- Permite efetuar deploy de aplicações executadas em *containers*
- Por exemplo, um código abaixo (`app.js`)
```javascript
const express = require('express');

const app = express();
app.use(express.json());

// Endpoint simples
app.get('/', (req, res) => {
  res.send('API rodando no Cloud Run!');
});

// Criar pedido
app.post('/pedido', (req, res) => {
  const { cliente, valor } = req.body;

  if (!cliente || !valor) {
    return res.status(400).json({ erro: 'Dados inválidos' });
  }

  res.json({
    mensagem: 'Pedido recebido',
    pedido: {
      cliente,
      valor,
      criadoEm: new Date()
    }
  });
});

const PORT = process.env.PORT || 8080;
app.listen(PORT, () => {
  console.log(`Servidor rodando na porta ${PORT}`);
});
```
- Criar um arquivo de requisitos `package.json`
```json
{
  "name": "cloud-run-pedidos",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^5.2.1"
  }
}
```
- Montar um arquivo `Dockerfile` para criação da imagem e *container*
```yaml
FROM node:20

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

ENV PORT=8080

CMD ["npm", "start"]
```
- Habilitar a API
```bash
gcloud services enable run.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com
```
- Criar um repositório de artefatos para armazenar a imagem do container
```bash
gcloud artifacts repositories create cloud-run-source-deploy --repository-format=docker --location=southamerica-east1 --description="Repositorio Docker para Cloud Run"

gcloud artifacts repositories list
```
- Isso irá criar um repositório com uma URL `southamerica-east1-docker.pkg.dev/SEU_PROJECT_ID/cloud-run-source-deploy`
- Criar uma imagem com base nas configurações
```bash
gcloud builds submit --tag southamerica-east1-docker.pkg.dev/ssa-$USER/cloud-run-source-deploy/hello-cloud-run
```
- Efetuar o *deploy* com base na imagem
```bash
gcloud run deploy hello-cloud-run --image southamerica-east1-docker.pkg.dev/ssa-$USER/cloud-run-source-deploy/hello-cloud-run --region southamerica-east1 --allow-unauthenticated
```
- Para visualizar as execuções acessar a URL [https://console.cloud.google.com/run](https://console.cloud.google.com/run)
#### App Engine
- Serviço **PAAS** para publicação de aplicações
- Criar o **App Engine**
```bash
gcloud app create --region=southamerica-east1

gcloud app describe
```
- Criar o diretório da aplicação
```bash
mkdir app-engine-demo

cd app-engine-demo
```
- Definir o `package.json` (no caso de uma aplicação **nodejs**)
```bash
cat <<EOF > package.json
{
  "name": "app-engine-demo",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  }
}
EOF
```
- Criar a aplicação
```javascript
cat <<EOF > app.js
const http = require('http');

const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
        mensagem: "Olá do Google App Engine"
    }));
});

const PORT = process.env.PORT || 8080;
server.listen(PORT, () => {
    console.log('Servidor rodando...', PORT);
});
EOF
```
- Criar o arquivo de *deploy*
```bash
cat <<EOF > app.yaml
runtime: nodejs22
entrypoint: node app.js
instance_class: F1
env_variables:
  NODE_ENV: development
automatic_scaling:
  min_instances: 1
  max_instances: 3
  target_cpu_utilization: 0.6
EOF
```
- Comando para efetuar o *deploy* da aplicação
```bash
gcloud app deploy

gcloud app deploy --no-cache --verbosity=debug --version=v1

gcloud app operations list

gcloud app browse

gcloud app logs tail -s default

gcloud app versions list

gcloud app versions delete VERSION_ID

gsutil ls gs://staging.app-project-esensato-1.appspot.com/*
```
- Para verificar as aplicações **app engine** [https://console.cloud.google.com/appengine](https://console.cloud.google.com/appengine)
### Banco Dados Firebase
- Criar o projeto e instalar as dependências
```bash
mkdir app-pedido
cd app-pedido
npm init -y
npm install --save express @google-cloud/firestore
```
- Criar uma instância do banco de dados
```bash
gcloud firestore databases create --location=southamerica-east1 --database app-pedido-$USER
```
- Código exemplo
```javascript
const express = require('express');
const { Firestore } = require('@google-cloud/firestore');

const app = express();
app.use(express.json());

const db = new Firestore({
  databaseId: '<COLOCAR_ID_AQUI_app-pedido-$USER>'
});

// Criar pedido
app.post('/pedido', async (req, res) => {
    const { cliente, valor } = req.body;

    const docRef = await db.collection('pedidos').add({
        cliente,
        valor,
        criadoEm: new Date()
    });

    res.json({ id: docRef.id });
});

// Listar pedidos
app.get('/pedidos', async (req, res) => {
    const snapshot = await db.collection('pedidos').get();

    const pedidos = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
    }));

    res.json(pedidos);
});

const PORT = process.env.PORT || 8080;
app.listen(PORT, () => console.log(`Rodando na porta ${PORT}`));
```
- Obter a URL de acesso
```bash
gcloud app browse
```
- Testar utilizando o `curl`
```bash
curl -X POST https://SEU_URL/pedido -H "Content-Type: application/json" -d '{"cliente":"João","valor":100}'
```
- Para verificar as instâncias do **firestore** [https://console.cloud.google.com/firestore/databases](https://console.cloud.google.com/firestore/databases)
### Google Cloud Functions
- Criação de funções *serveless*
- Permissões necessárias
```bash
gcloud services enable pubsub.googleapis.com
gcloud services enable firestore.googleapis.com
```
- Código exemplo (arquivo `index.js`)
```javascript
exports.helloHttp = (req, res) => {
  const nome = req.query.nome || "mundo";
  res.status(200).send(`Olá, ${nome}!`);
};
```
```bash
gcloud functions deploy helloHttp --runtime=nodejs22 --trigger-http --allow-unauthenticated
```
- Para testar: `curl "https://REGION-PROJECT.cloudfunctions.net/helloHttp?nome=Edson"`
- Exemplo com uma requisição *POST*
```javascript
exports.criarPedido = (req, res) => {
  const { cliente, valor } = req.body;

  if (!cliente || !valor) {
    return res.status(400).json({ erro: "Dados inválidos" });
  }

  res.status(201).json({
    mensagem: "Pedido criado",
    pedido: {
      cliente,
      valor,
      criadoEm: new Date()
    }
  });
};
```
- Para testar com *curl*
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"cliente":"João","valor":100}' \
  https://REGION-PROJECT.cloudfunctions.net/criarPedido
```
- Exemplo *pub / sub*
```javascript
exports.processarPedido = (event) => {
  const mensagem = JSON.parse(
    Buffer.from(event.data, 'base64').toString()
  );

  console.log("Cliente:", mensagem.cliente);
  console.log("Valor:", mensagem.valor);
};
```
- Para publicar na fila
```bash
gcloud pubsub topics publish pedidos-topic \
  --message='{"cliente":"Maria","valor":250}'
```
- Exemplo combinando os dois tipos de *functions*
```bash
const { PubSub } = require('@google-cloud/pubsub');
const pubsub = new PubSub();

exports.receberPedido = async (req, res) => {
  const dados = JSON.stringify(req.body);

  await pubsub.topic('pedidos-topic').publishMessage({
    data: Buffer.from(dados)
  });

  res.status(200).send("Pedido enviado para processamento");
};
```
- Exemplo de integração com o **Firestore**
```javascript
const { Firestore } = require('@google-cloud/firestore');

const db = new Firestore();

exports.processarPedido = async (event) => {
  try {
    const mensagem = JSON.parse(
      Buffer.from(event.data, 'base64').toString()
    );

    console.log("Pedido recebido:", mensagem);

    const docRef = await db.collection('pedidos').add({
      cliente: mensagem.cliente,
      valor: mensagem.valor,
      criadoEm: mensagem.criadoEm,
      processadoEm: new Date().toISOString()
    });

    console.log("Pedido salvo com ID:", docRef.id);

  } catch (erro) {
    console.error("Erro ao processar pedido:", erro);
    throw erro; // importante para retry automático
  }
};
```
### Google Kubernetes Engine
- Habilitar API
```bash
gcloud services enable container.googleapis.com
```
- Criar o cluster
```bash
gcloud container clusters create-auto aula-gke --region southamerica-east1
```
- Conectar ao cluster
```bash
gcloud container clusters get-credentials aula-gke --region southamerica-east1
```
- Listar os PODs
```bash
kubectl get nodes
```
- Criar o primeiro *deployment*
```bash
kubectl create deployment nginx-demo --image=nginx:1.27
kubectl get pods
```
- Obter o *service*
```bash
kubectl expose deployment nginx-demo \
  --port=80 \
  --target-port=80 \
  --type=LoadBalancer
```
- Obter o IP público e testar o serviço em um navegador
```bash
kubectl get service nginx-demo
```
- Escalar réplicas
```bash
kubectl scale deployment nginx-demo --replicas=3
kubectl get pods -o wide
```
- Remover um POD e verificar sua recriação
```bash
kubectl delete pod NOME_DO_POD
kubectl get pods -o wide
```
- Efetuar um *roll-out**
```bash
kubectl set image deployment/nginx-demo \
  nginx=nginx:latest

kubectl rollout status deployment/nginx-demo

kubectl describe deployment nginx-demo

kubectl logs deployment/nginx-demo
```
- Gerar o arquivo *yaml* do *deployment*
```bash
kubectl get deployment nginx-demo -o yaml
```
- Para aplicar um *manifest*
```bash
kubectl apply -f nginx.yaml
```
- Comandos úteis
```bash
kubectl get all
kubectl get deployment
kubectl get replicaset
kubectl get pods
kubectl get services
```
- Removendo os recursos
```bash
kubectl delete service nginx-demo
kubectl delete deployment nginx-demo
gcloud container clusters delete aula-gke --region southamerica-east1
```
#### ConfigMaps
- Criar um *ConfigMap*
```bash
kubectl create configmap app-config --from-literal=MENSAGEM="Olá do ConfigMap no Kubernetes!"
```
- Verificar o *ConfigMap* criado
```bash
kubectl get configmap
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```
- Criar o *deployment* que irá utilizar o *ConfigMap* (arquivo `deployment-configmap.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configmap-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: configmap-demo
  template:
    metadata:
      labels:
        app: configmap-demo
    spec:
      containers:
      - name: app
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            echo "$MENSAGEM" > /www/index.html
            httpd -f -p 8080 -h /www
            sleep 3600
          done
        env:
        - name: MENSAGEM
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: MENSAGEM
        ports:
        - containerPort: 8080
```
- Aplicar o *deployment*
```bash
kubectl apply -f deployment-configmap.yaml
```
- Expor o serviço e obter o IP público
```bash
kubectl expose deployment configmap-demo \
  --port=80 \
  --target-port=8080 \
  --type=LoadBalancer

kubectl get service configmap-demo
```
- Limpando os recursos alocados
```bash
kubectl delete service configmap-demo
kubectl delete deployment configmap-demo
kubectl delete configmap app-config
```
#### Secrets
- Criar um *secret*
```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=MinhaSenhaSuperSecreta
```
- Visualizar o *secret*
```bash
kubectl get secrets
kubectl describe secret db-secret
kubectl get secret db-secret -o yaml
```
- Criar o *deployment* que irá utilizar o *secret* (arquivo `secret-demo.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secret-demo
  template:
    metadata:
      labels:
        app: secret-demo
    spec:
      containers:
      - name: app
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Usuário: $DB_USER"
          echo "Senha: $DB_PASSWORD"
          sleep 3600
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
```
- Aplicar o *deployment*
```bash
kubectl apply -f secret-demo.yaml
```
- Verificar os valores no *log*
```bash
kubectl logs deployment/secret-demo
```
- Verificar os valores dentro do próprio *container*
```bash
kubectl exec -it deployment/secret-demo -- sh

echo $DB_USER
echo $DB_PASSWORD
```
- Para atualizar o *secret*
```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=NovaSenha123 \
  --dry-run=client -o yaml | kubectl apply -f -
```
- Aplicar a atualização para os *PODs*
```bash
kubectl rollout restart deployment secret-demo
```
- Liberando os recursos alocados
```bash
kubectl delete deployment secret-demo
kubectl delete secret db-secret
```
#### Volumes
- Exemplo de *emptyDir* (arquivo `volume-demo.yaml`)
- Neste exemplo, um *container* irá escrever no arquivo e o outro irá ler
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
  - name: writer
    image: busybox
    command:
    - sh
    - -c
    - |
      i=0
      while true; do
        echo "Contador: $i" > /dados/contador.txt
        i=$((i+1))
        sleep 2
      done
    volumeMounts:
    - name: dados
      mountPath: /dados

  - name: reader
    image: busybox
    command:
    - sh
    - -c
    - |
      while true; do
        cat /dados/contador.txt 2>/dev/null || echo "Arquivo ainda não existe"
        sleep 2
      done
    volumeMounts:
    - name: dados
      mountPath: /dados

  volumes:
  - name: dados
    emptyDir: {}
```
- Aplicar o *deployment*
```bash
kubectl apply -f volume-demo.yaml
```
- Exibir os *logs*
```bash
kubectl logs volume-demo -c reader -f
```
- Reiniciar os *containers* e verificar no *log* que o contador é zerado
```bash
kubectl delete pod volume-demo
kubectl apply -f volume-demo.yaml

kubectl logs volume-demo -c reader -f
```
- Definindo um *volume* para *ConfigMap*
```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
```
- Definindo um *volume* para *secret*
```yaml
volumes:
- name: secret-volume
  secret:
    secretName: db-secret
```
- Exemplo de *PersistentVolumeClaim* (arquivo `pvc-demo.yaml`)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: contador-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvc-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pvc-demo
  template:
    metadata:
      labels:
        app: pvc-demo
    spec:
      containers:
      - name: app
        image: busybox
        command:
        - sh
        - -c
        - |
          if [ ! -f /dados/contador ]; then
            echo 0 > /dados/contador
          fi

          while true; do
            valor=$(cat /dados/contador)
            echo "Valor atual: $valor"
            echo $((valor + 1)) > /dados/contador
            sleep 5
          done
        volumeMounts:
        - name: dados
          mountPath: /dados
      volumes:
      - name: dados
        persistentVolumeClaim:
          claimName: contador-pvc
```
- Aplicar
```bash
kubectl apply -f pvc-demo.yaml
```
- Verificar o *pvc* e o *pv* criados
```bash
kubectl get pvc
kubectl get pv
```
- Verificar os *logs*
```bash
kubectl logs deployment/pvc-demo -f
```
- Remover o *POD*
```bash
kubectl delete pod -l app=pvc-demo
```
- Verificar os *logs*
```bash
kubectl logs deployment/pvc-demo -f
```
- Verificar os *storage class*
```bash
kubectl get storageclass
```
