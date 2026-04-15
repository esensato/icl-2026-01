### AV-02-ICL-Azure-Functions

#### Preparação

- Acessar o [portal.azure.com](https://portal.azure.com/auth/login/)
- Já no portal, acessar o **Cloud Shell** na barra superior da tela
- Cria um grupo de recursos chamado `av-02-icl-azure-functions` na *location* `brazilsouth`
- Instanciar o serviço de reconhecimento de documentos
```bash
az cognitiveservices account create --name doc-intelligence --resource-group av-02-icl-azure-functions --kind FormRecognizer --sku F0 --location brazilsouth --yes
```
- Obter o *endpoint* pra acesso
```bash
az cognitiveservices account show --name doc-intelligence --resource-group av-02-icl-azure-functions --query properties.endpoint -o tsv
```
- Obter a chave de API
```bash
az cognitiveservices account keys list --name doc-intelligence --resource-group av-02-icl-azure-functions
```
- Criar uma aplicação do tipo **Azure Functions**
```bash
$env:AZURE_CORE_ONLY_SHOW_ERRORS = "true"

az storage account create --name av02iclstorage --resource-group av-02-icl-azure-functions --location brazilsouth --sku Standard_LRS

az functionapp create --resource-group av-02-icl-azure-functions --consumption-plan-location brazilsouth --runtime node --functions-version 4 --name app-functions-$(Get-Date -Format 'yyyyMMddHHmmss') --storage-account av02iclstorage
```
- Criar uma instância de banco de dados **SQL Server**
``` bash
az sql server create --name av02iclsql --resource-group av-02-icl-azure-functions --location brazilsouth --admin-user adminuser --admin-password "SenhaForte!123"

az sql server firewall-rule create --resource-group av-02-icl-azure-functions --server av02iclsql --name AllowAllIP --start-ip-address 0.0.0.0 --end-ip-address 255.255.255.255

az sql db create --resource-group av-02-icl-azure-functions --server av02iclsql --name db --service-objective Basic --yes

az sql db list --resource-group av-02-icl-azure-functions --server av02iclsql --output table
```
- Efretuar a conexão com o banco de dados criado
```bash
sqlcmd -S av02iclsql.database.windows.net -d db -U adminuser -P 'SenhaForte!123'
```
- Código *SQL* para criar as tabelas utilizadas nos exemplos
```sql
CREATE TABLE RECIBO (Id INT IDENTITY(1,1) NOT NULL PRIMARY KEY, TOTAL NVARCHAR(20));
GO
```
- Criar a variável com a *string* de conexão com o banco de dados na **Azure**
```bash
az functionapp config appsettings set --name app-functions-<<OBTER_NOME_FUNCAO>> --resource-group av-02-icl-azure-functions --settings SqlConnectionString="Server=tcp:av02iclsql.database.windows.net,1433;Initial Catalog=db;User ID=adminuser;Password=SenhaForte!123;Encrypt=True;"
```
- Para testes locais incluir no arquivo `local.settings.json`
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "SqlConnectionString": "Server=tcp:av02iclsql.database.windows.net,1433;Initial Catalog=db;User ID=adminuser;Password=SenhaForte!123;Encrypt=True;",
    "FUNCTIONS_WORKER_RUNTIME": "node"
  }
}
```
#### VS Code
- Instalar as dependências
```bash
npm install --save parse-multipart-data axios form-data fs
```
- Obter a URL pública de acesso à função após do *deploy* na **Azure**
```bash
az functionapp show --name app-functions-<<OBTER_NOME_FUNCAO>> --resource-group av-02-icl-azure-functions --query defaultHostName -o tsv
```
- Com base no retorno acima, para chamar a função criada ficaria: `https://<URL_BASE_ACIMA>/api/nome_da_funcao_criada_vscode`
- Criar uma rotina clinte para testar a aplicação
```javascript
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');

async function enviarImagem() {

    const URL = "https://app-functions-20260415184656.azurewebsites.net/api/analisarupload";
    // const URL = "http://localhost:7071/api/analisarUpload";
    const form = new FormData();

    form.append(
        'file',
        fs.createReadStream('./contoso-receipt.png')
    );

    const response = await axios.post(
        URL,
        form,
        {
            headers: form.getHeaders()
        }
    );

    return response.data.resultado.fields;

}


const teste = async () => {

    const resposta = await enviarImagem();
    console.log(resposta);

}

teste();
```
#### Implementação
- Criar uma função do tipo **HTTP** que deve receber um arquivo de imagem e encaminhar para análise do serviço cognitivo instanciado anteriormente
- Utilizar o código de base
```javascript

  const CHAVE_API_KEY = <<CHAVE_DE_API_OBTIDA_ANTERIORMENTE>>
  
  function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
  
  try {
  
  // Pega body como buffer
  const bodyBuffer = Buffer.from(await request.arrayBuffer());
  
  // Extrai boundary do multipart
  const contentType = request.headers.get('content-type');
  const boundary = contentType.split('boundary=')[1];
  
  // Parse multipart
  const parts = multipart.parse(bodyBuffer, boundary);
  
  if (!parts.length) {
      return {
          status: 400,
          jsonBody: { erro: "Nenhum arquivo enviado." }
      };
  }
  
  const arquivo = parts[0];
  
  const endpoint = "https://brazilsouth.api.cognitive.microsoft.com/";
  
  console.log('Enviando para processamento')
  const response = await axios.post(
      `${endpoint}documentintelligence/documentModels/prebuilt-receipt:analyze?api-version=2024-11-30`,
      arquivo.data,
      {
          headers: {
              'Ocp-Apim-Subscription-Key': CHAVE_API_KEY,
              'Content-Type': 'application/octet-stream'
          }
      }
  );
  
  console.log(`Imagem em processamento: ${response.headers['operation-location']}`);
  
  console.log('Obtendo o resultado');
  
  let analise = await axios.get(
      response.headers['operation-location'],
      {
          headers: {
              'Ocp-Apim-Subscription-Key': CHAVE_API_KEY
          }
      }
  );
  
  while (analise.data.status === 'running') {
      analise = await axios.get(
          response.headers['operation-location'],
          {
              headers: {
                  'Ocp-Apim-Subscription-Key': CHAVE_API_KEY
              }
          }
      );
  
      console.log(analise.data.status);
  
      await sleep(5000);
  }
  
  console.log(analise.data.analyzeResult.documents[0]);
  
  return {
      status: 200,
      jsonBody: {
        resultado: analise.data.analyzeResult.documents[0].fields
      }
  };
  
  } catch (err) {
  
  return {
      status: 500,
      jsonBody: {
          erro: err.message
      }
  };
  }
```
- A função deve extrair o atributo `Total` retornado (`analise.data.analyzeResult.documents[0].fields.Total`) e encaminhar para uma fila
- O consumidor desta fila deve inserir o total em uma tabela do banco de dados (criado anteriormente)
- Criar uma nova função que responda à requisições `GET` para listar todos os totais armazenados até o momento na tabela

