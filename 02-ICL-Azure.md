## Microsoft Azure
- Obter créditos **Student** para no [azure.microsoft.com](https://azure.microsoft.com/en-us/free/students/?culture=pt-br&country=br) *(pressionar control para abrir em nova página)*
- Acessar o [portal.azure.com](https://portal.azure.com/auth/login/)
```bash
az group list --output table
az resource list --output table

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
- Queue
```javascript
const { app } = require('@azure/functions');

app.storageQueue('queueFunction', {
    queueName: 'minha-fila',
    connection: 'AzureWebJobsStorage',

    handler: async (message, context) => {
        context.log('Mensagem:', message);
    }
});
```
- Blob
```javascript
const { app } = require('@azure/functions');

app.storageBlob('blobFunction', {
    path: 'arquivos/{name}',
    connection: 'AzureWebJobsStorage',

    handler: async (blob, context) => {
        context.log('Arquivo processado');
    }
});
```
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
            body: JSON.stringify({ msg: "ok" })
        };
    }
});
```
- Editar o arquivo `local.settings.json` e incluir a configuração para acesso ao banco de dados `SqlConnectionString` dentro de `Values`

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
