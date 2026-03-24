## Microsoft Azure
- Obter créditos **Student** para no [azure.microsoft.com](https://azure.microsoft.com/en-us/free/students/?culture=pt-br&country=br) *(pressionar control para abrir em nova página)*

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
### Functions
- Pressionar `Shift` + `Control` + `P` no **VS Code**
- Digitar `Create function...`
```bash
npm install @azure/msal-node express express-session
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
