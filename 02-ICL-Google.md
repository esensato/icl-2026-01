## Google Cloud
- Acesso ao console web https://console.cloud.google.com/
)
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
- Código exemplo (arquivo `index.js`)
```javascript
exports.helloHttp = (req, res) => {
  res.send("Hello Functions!");
};
```
```bash
gcloud functions deploy helloHttp --runtime=nodejs22 --trigger-http --allow-unauthenticated
```
