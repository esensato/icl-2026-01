## Google Cloud

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
```
- Criar um projeto e definí-lo no contexto (projeto de trabalho atual)
```bash
gcloud projects create app-project-$USER

gcloud config set project app-project-$USER
```
- Acessar o console
```bash
gcloud config get-value project
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
```
#### App Engine
- Serviço **PAAS** para publicação de aplicações
- Criar o **App Engine**
```bash
gcloud app create --region=southamerica-east1

gcloud app describe
```
- Verificar os **buckets** criados
```bash
gcloud storage buckets list | grep gs
gcloud iam service-accounts list
gcloud services list --enabled

```
- Adicionar as permissões necessárias (observar `defaultBucket` e `serviceAccount`)
```bash
gcloud storage buckets add-iam-policy-binding gs://<DEFAULT_BUKET_COMANDO_ANTERIOR> --role="roles/storage.objectAdmin" --member="serviceAccount:<OBTIDO_COMANDO_ANTERIOR>" 
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

gsutil rm -r gs://staging.app-project-esensato-1.appspot.com/*
```
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
- Criar uma VM
```bash
gcloud compute instances create vm-demo --zone=us-central1-a --machine-type=e2-micro --image-family=ubuntu-2204-lts --image-project=ubuntu-os-cloud
```


https://console.cloud.google.com/firestore/databases

https://console.cloud.google.com/appengine

gcloud app logs tail -s default

### Google Cloud Functions
- Criação de funções *serveless*
- Código exemplo (arquivo `index.js`)
```javascript
exports.helloHttp = (req, res) => {
  res.send("Hello Functions!");
};
```
```bash
gcloud functions deploy helloHttp --runtime=nodejs22 --trigger-http --allow-unauthenticated
```
