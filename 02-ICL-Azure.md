## Microsoft Azure
- Obter créditos **Student** para no [azure.microsoft.com](https://azure.microsoft.com/en-us/free/students/?culture=pt-br&country=br) *(pressionar control para abrir em nova página)*
- Acessar o [portal.azure.com](https://portal.azure.com/auth/login/)
```bash
az group list --output table
az resource list --output table
az provider register --namespace Microsoft.OperationalInsights
```
### Acessando Banco SQL Server
```sql
CREATE TABLE [dbo].[Recibos] (
    Id INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    Cliente NVARCHAR(100) NULL,
    Total FLOAT NULL);
```
```bash
npm install --save mssql
```
```javascript
const sql = require("mssql");

const config = {
  user: "esensato@esensato",
  password: "SUA_SENHA",
  server: "esensato.database.windows.net",
  database: "db",
  port: 1433,
  options: {
    encrypt: true, // obrigatório no Azure
    trustServerCertificate: false
  },
  connectionTimeout: 30000
};

async function conectar() {
  try {
    await sql.connect(config);

    const result = await sql.query("SELECT GETDATE() as data");

    console.log(result.recordset);

  } catch (err) {
    console.error("Erro:", err);
  }
}

conectar();
```
```javascript
async function inserirRecibo() {
  try {
    await sql.connect(config);

    const request = new sql.Request();

    request.input("Id", sql.Int, 1);
    request.input("Cliente", sql.NVarChar(100), "João Silva");
    request.input("Total", sql.Float, 150.75);

    await request.query(`
      INSERT INTO dbo.Recibos (Id, Cliente, Total)
      VALUES (@Id, @Cliente, @Total)
    `);

    console.log("✅ Registro inserido com sucesso!");

  } catch (err) {
    console.error("❌ Erro:", err);
  } finally {
    sql.close();
  }
}
```
### Document Intelligence API
```bash
npm init -y
npm install axios express multer
```
```javascript
const express = require("express");
const multer = require("multer");
const axios = require("axios");
const fs = require("fs");

const app = express();
const upload = multer({ dest: "uploads/" });

const PORT = process.env.PORT || 3000;

const AZURE_KEY = "SUA_CHAVE";
const AZURE_ENDPOINT = "SEU_ENDPOINT"; 

app.get("/", (req, res) => {
  res.send(`
    <h2>Upload de documento (PDF/Imagem)</h2>
    <form method="POST" action="/analyze" enctype="multipart/form-data">
      <input type="file" name="file"/>
      <button type="submit">Enviar</button>
    </form>
  `);
});

// Endpoint principal
app.post("/analyze", upload.single("file"), async (req, res) => {
  try {
    const fileData = fs.readFileSync(req.file.path);

    const response = await axios.post(
      `${AZURE_ENDPOINT}/formrecognizer/documentModels/prebuilt-receipt:analyze?api-version=2023-07-31`,
      fileData,
      {
        headers: {
          "Ocp-Apim-Subscription-Key": AZURE_KEY,
          "Content-Type": "application/octet-stream"
        }
      }
    );

    const operationLocation = response.headers["operation-location"];

    let result;
    while (true) {
      const poll = await axios.get(operationLocation, {
        headers: {
          "Ocp-Apim-Subscription-Key": AZURE_KEY
        }
      });

      if (poll.data.status === "succeeded") {
        result = poll.data;
        break;
      }

      if (poll.data.status === "failed") {
        throw new Error("Falha na análise");
      }

      await new Promise(r => setTimeout(r, 2000));
    }

    fs.unlinkSync(req.file.path);

    res.json(result);

  } catch (err) {
    console.error(err.response?.data || err.message);
    res.status(500).send("Erro ao analisar documento");
  }
});

app.listen(PORT, () => {
  console.log("Servidor rodando na porta", PORT);
});
```
### Azure Functions
- Para efetuar testes locais
```bash
npm install -g azure-functions-core-tools@4
npm install @azure/functions
npm install @azure/functions-extensions-azure-sql

func start
```
- HTTP Funcion
```javascript
const { app } = require('@azure/functions');

app.http('helloFunction', {
    methods: ['GET', 'POST'],
    authLevel: 'anonymous',
    handler: async (request, context) => {
        context.log(`Http function processed request for url "${request.url}"`);

        const name = request.query.get('name') || await request.text() || 'world';

        return { body: `Hello, ${name}!` };
    }
});
```
- Timer Function
```javascript
const { app } = require('@azure/functions');

app.timer('timerFunction', {
    schedule: '*/10 * * * * *',

    handler: async (myTimer, context) => {
        context.log('Executando a cada 10 segundos');
    }
});
```
- Queue produtor
```javascript
const { app, output } = require('@azure/functions');

// Output binding para fila
const queueOutput = output.storageQueue({
    queueName: 'minha-fila',
    connection: 'AzureWebJobsStorage'
});

app.http('enviarMensagemFunction', {
    methods: ['POST'],
    authLevel: 'anonymous',

    extraOutputs: [queueOutput],

    handler: async (request, context) => {

        const body = await request.json();

        const mensagem = {
            cliente: body.cliente,
            total: body.total,
            data: new Date().toISOString()
        };

        // Envia para fila
        context.extraOutputs.set(queueOutput, mensagem);

        context.log("Mensagem enviada para fila:", mensagem);

        return {
            status: 200,
            body: {
                message: "Mensagem enviada com sucesso",
                data: mensagem
            }
        };
    }
});
```
- Queue consumidor
```javascript
const { app } = require('@azure/functions');

app.storageQueue('processarFilaFunction', {
    queueName: 'minha-fila',
    connection: 'AzureWebJobsStorage',

    handler: async (message, context) => {

        context.log("Mensagem recebida da fila:");

        context.log(message);

        // Exemplo de processamento
        context.log(`Cliente: ${message.cliente}`);
        context.log(`Total: ${message.total}`);

        // Aqui você poderia:
        // - salvar no banco
        // - chamar outra API
        // - enviar email
    }
});
```
- Blob
- Código **Nodejs** cliente para efetuar o upload do arquivo
- Criar o projeto
```bash
mkdir upload-blob
cd upload-blob
npm init -y
```
- Instalar o **Azurite** para testar o armazenamento localmente
```bash
npm install -g azurite
azurite --skipApiVersionCheck
```
- Instalar as dependências
```bash
npm install --save @azure/storage-blob express multer
```
- Criar o arquivo para *upload* (no caso, o arquivo considerado se chama `arquivo.txt`)
- Implementar o código para efetuar o *upload*
```javascript
const { BlobServiceClient } = require('@azure/storage-blob');
const fs = require('fs');

const connectionString = "UseDevelopmentStorage=true";
const containerName = "uploads";
const filePath = "./arquivo.txt";

async function uploadBlob() {
    try {
        const blobServiceClient = BlobServiceClient.fromConnectionString(connectionString);
        const containerClient = blobServiceClient.getContainerClient(containerName);
        await containerClient.createIfNotExists();
        // Somente para teste (falha de segurança pois permite o acesso via URL direto ao arquivo!)
        await containerClient.setAccessPolicy('blob');
        const blobName = "arquivo-" + Date.now() + ".txt";
        const blockBlobClient = containerClient.getBlockBlobClient(blobName);
        const uploadResponse = await blockBlobClient.uploadFile(filePath);
        console.log("Upload realizado com sucesso!", JSON.stringify(uploadResponse));
        console.log("Blob:", blobName);

    } catch (err) {
        console.error("Erro:", err.message);
    }
}

uploadBlob();
```
- O arquivo pode ser acessado por meio da URL `http://127.0.0.1:10000/devstoreaccount1/uploads/NOME_DO_ARQUIVO` (trocar o `NOME_DO_ARQUIVO`)
- Quando um arquivo é carregado (*upload*) então uma função do tipo `storageBlob` pode ser disparada
```javascript
const { app } = require('@azure/functions');

app.storageBlob('processarArquivoFunction', {
    path: 'uploads/{name}',
    connection: 'AzureWebJobsStorage',

    handler: async (blob, context) => {

        const fileName = context.triggerMetadata.name;

        context.log(`Arquivo recebido: ${fileName}`);
        context.log(`Tamanho: ${blob.length} bytes`);
        const content = blob.toString();

        context.log("Conteúdo:", content);
    }
});
```
- Para o ambiente de cloud, consultar o [Storage Account](https://portal.azure.com/#view/Microsoft_Azure_StorageHub/StorageHub.MenuView/~/StorageAccountsBrowse)
```bash
az storage account keys list --resource-group <resource-group> --account-name <storage-account>
```
#### Exemplo de uma aplicação com *frontend* para o upload de imagem
- Criar uma pasta `public` no projeto para conter os arquivos *HTML* e *CSS*
- Código HTML para enviar a imagem (index.html)
```html
<!DOCTYPE html>
<html>
<head>
    <title>Upload de Imagem</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>

<div class="container">
    <h1>Upload de Imagem</h1>

    <form action="/upload" method="POST" enctype="multipart/form-data">
        <input type="file" name="file" required>
        <button type="submit">Enviar</button>
    </form>
</div>

</body>
</html>
```
- Código HTML para exibir a imagem (view.html)
```html
<!DOCTYPE html>
<html>
<head>
    <title>Imagem Enviada</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>

<div class="container">
    <h1>Upload realizado!</h1>
    <img id="preview" />
    <br><br>
    <a href="/">Voltar</a>
</div>

<script>
    const params = new URLSearchParams(window.location.search);
    const img = params.get("img");

    document.getElementById("preview").src = "/uploads/" + img;
</script>

</body>
</html>
```
- Arquivo de estilos (`style.css`)
```css
body {
    font-family: Arial;
    background: #f2f2f2;
    display: flex;
    justify-content: center;
    margin-top: 100px;
}

.container {
    background: white;
    padding: 30px;
    border-radius: 10px;
    box-shadow: 0 0 10px #ccc;
    text-align: center;
}

h1 {
    margin-bottom: 20px;
}

input[type="file"] {
    margin-bottom: 15px;
}

button {
    background: #0078d4;
    color: white;
    border: none;
    padding: 10px 20px;
    border-radius: 5px;
    cursor: pointer;
}

button:hover {
    background: #005fa3;
}

img {
    max-width: 300px;
    border-radius: 10px;
    box-shadow: 0 0 10px #ccc;
}
```
- Código **nodejs** para processar a imagem (`server.js`)
```javascript
const express = require('express');
const multer = require('multer');
const path = require('path');

const app = express();

const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, 'uploads/');
    },
    filename: (req, file, cb) => {
        cb(null, file.originalname);
    }
});

// Pasta para salvar uploads
const upload = multer({ storage });

// Servir arquivos estáticos
app.use(express.static('public'));
app.use('/uploads', express.static('uploads'));

// Rota de upload
app.post('/upload', upload.single('file'), (req, res) => {
    const file = req.file;

    // Redireciona para página de visualização
    res.redirect(`/view.html?img=${file.filename}`);
});

// Iniciar servidor
app.listen(3000, () => {
    console.log('Servidor rodando em http://localhost:3000');
});
```
- Adaptar o código acima para realizar o upload na **Azure**
### Acesso Banco de Dados
- Exemplo de uma function que utiliza o recurso de **binding** para efetuar uma consulta ao banco de dados
```javascript
const { app, input } = require('@azure/functions');

// Define o binding de saída (SQL)
const sqlInput = input.sql({
    commandText: 'SELECT Id, Cliente, Total FROM dbo.Recibos',
    connectionStringSetting: 'SqlConnectionString'
});

app.http('listarRecibosFunction', {
    methods: ['GET'],
    authLevel: 'anonymous',
    extraInputs: [sqlInput],
    handler: async (request, context) => {

        context.log("Buscando recibos no banco...");

        context.log(sqlInput);

        // Executa o binding automaticamente
        const recibos = context.extraInputs.get(sqlInput);

        context.log("Resultado: ", recibos);

        return {
            status: 200,
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(recibos)
        };
    }
});
```
- Editar o arquivo `local.settings.json` e incluir a configuração para acesso ao banco de dados `SqlConnectionString` dentro de `Values`
- Exemplo com passagem de parâmetros
```javascript
const { app, input } = require('@azure/functions');

// Binding com parâmetro
const sqlInput = input.sql({
    commandText: `
        SELECT Id, Cliente, Total 
        FROM dbo.Recibos
        WHERE Cliente = @cliente
    `,
    parameters: [
        {
            name: 'cliente',
            type: 'nvarchar',
            value: '{query.cliente}'
        }
    ],
    connectionStringSetting: 'SqlConnectionString'
});

app.http('listarRecibosFunction', {
    methods: ['GET'],
    authLevel: 'anonymous',

    extraInputs: [sqlInput],

    handler: async (request, context) => {

        context.log("Buscando recibos por cliente...");

        const cliente = request.query.get('cliente');
        context.log("Cliente recebido:", cliente);

        const recibos = context.extraInputs.get(sqlInput);

        return {
            status: 200,
            body: recibos
        };
    }
});
```
- Para inserir um registro
```javascript
const { app, output } = require('@azure/functions');

// Binding de saída (INSERT)
const sqlOutput = output.sql({
    commandText: 'dbo.Recibos', // nome da tabela
    connectionStringSetting: 'SqlConnectionString'
});

app.http('inserirReciboFunction', {
    methods: ['POST'],
    authLevel: 'anonymous',

    extraOutputs: [sqlOutput],

    handler: async (request, context) => {

        context.log("Inserindo novo recibo...");

        // Pegando dados do body (JSON)
        const body = await request.json();

        const novoRecibo = {
            Cliente: body.cliente,
            Total: body.total
        };

        // Envia para o binding (INSERT automático)
        context.extraOutputs.set(sqlOutput, [novoRecibo]);

        context.log("Recibo inserido:", novoRecibo);

        return {
            status: 201,
            headers: {
                'Content-Type': 'application/json'
            },
            body: {
                message: "Recibo inserido com sucesso",
                data: novoRecibo
            }
        };
    }
});
```
- Como ficaria uma aplicação que chama o *endpoint* para cadastrar um recibo (atualizar a **URL** com o endereço gerado para publicação na **Azure**)
```javascript
async function inserirRecibo() {
    const URL_POST = 'http://localhost:7071/api/inserirReciboFunction';
    const response = await fetch(URL_POST, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            cliente: 'João da Silva',
            total: 250.90
        })
    });

    const data = await response.json();

    console.log("Resposta:", data);
}

inserirRecibo();
```
### Deploy Aplicação
- Efetuar login no [portal.azure.com](https://portal.azure.com/) *(pressionar control para abrir em nova página)*
- Abrir um **Cloud Shell** na barra de ferramentas superior dentro do **Portal Azure**
- Registrar *namespace* `Microsoft.OperationalInsights`
```bash
az provider register --namespace Microsoft.OperationalInsights
```
- Instalar o *plugin* **Azure Extensions** dentro do **VS Code**
- Efetuar o *login* via **VS Code** no **Azure** utilizando o mesmo usuário da faculdade (o mesmo que obteve os créditos)
- Criar um **App Service**
- Criar uma aplicação **Nodejs** de teste
```javascript
const express = require("express");

const app = express();

// IMPORTANTE: Azure define a porta dinamicamente
const port = process.env.PORT || 3000;

// Middleware para JSON
app.use(express.json());

// Rota principal
app.get("/", (req, res) => {
    res.send("<h1>🚀 App rodando no Azure com Express!</h1>");
});

// Rota de teste API
app.get("/api/hello", (req, res) => {
    res.json({
        message: "Hello World API",
        status: "ok"
    });
});

// Rota POST (exemplo)
app.post("/api/data", (req, res) => {
    res.json({
        received: req.body
    });
});

// Subir servidor
app.listen(port, () => {
    console.log(`Servidor rodando na porta ${port}`);
});
```
- Efetuar o *deploy* da aplicação na **Azure**
