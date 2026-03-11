### AV-01-ICL-Chatbot
- Siga o roteiro abaixo para implementar uma jornada para que alunos de uma universidade possam utilizar o **chatbot** para efetuar a matrícula em uma disciplina do curso:
    - Implementar uma **action** para que o aluno possa consultar seus créditos até o momento;
    - Implementar outra **action** para que o aluno possa efetuar a sua matrícula em uma disciplina, atualizando seus créditos conforme a disciplina selecionada;
#### Configurar DB2
- Será utilizado o serviço de banco de dados relacional **DB2** para esta atividade
- No catálogo da **IBM Cloud** pesquisar por **DB2** e instanciar o serviço (aguardar até que esteja provisionado) - [https://cloud.ibm.com/catalog#all_products](https://cloud.ibm.com/catalog#all_products)
- Na página dos recursos [https://cloud.ibm.com/resources](https://cloud.ibm.com/resources) aguardar o provisionamento e, quando concluído, clicar sobre **DB2** instanciado dentro do tópico **Databases**
- Na página do **DB2** clicar em **Credenciais de serviços** (menu lado esquerdo da tela)
- Clicar no botão **Criar credencial** (azul lado direito) e anotar:
    - `username`
    - `password`
- Clicar agora em **Gerenciar** (menu lado esquerdo da tela)
- Clicar no botão azul **Go to UI**
- No lado esquerdo, clicar no ícone **Chave** (Administração) e anotar:
    - `Nome do host da API de REST`
- Efetuar a carga dos arquivos abaixo nas respectivas tabelas:
    - **ALUNOS** - [https://github.com/esensato/icl-2026-01/blob/main/AV-01-ICL-alunos-db2.csv](https://github.com/esensato/icl-2026-01/blob/main/AV-01-ICL-alunos-db2.csv)
    - **DISCIPLINAS** - [https://github.com/esensato/icl-2026-01/blob/main/AV-01-ICL-disciplinas-db2.csv](https://github.com/esensato/icl-2026-01/blob/main/AV-01-ICL-disciplinas-db2.csv)
    - **MATRICULAS** - [https://github.com/esensato/icl-2026-01/blob/main/AV-01-ICL-matricula-db2.csv](https://github.com/esensato/icl-2026-01/blob/main/AV-01-ICL-matricula-db2.csv)
#### Watsonx Assistant
- No catálogo da **IBM Cloud** pesquisar por **Assistant** e instanciar o serviço (aguardar até que esteja provisionado) - [https://cloud.ibm.com/catalog#all_products](https://cloud.ibm.com/catalog#all_products)
- Importar o *template* [https://github.com/esensato/icl-2026-01/blob/main/assistente-modelo.zip](https://github.com/esensato/icl-2026-01/blob/main/assistente-modelo.zip)
- Clicar no serviço instanciado e acionar o botão **Launch watsonx Assistant**
- Criar as seguintes variáveis de sessão com os dados anotados e obtidos do **DB2**:
    - `DB2_DEPLOYMENT_ID`
    - `DB2_PASSWORD`
    - `DB2_TOKEN`
    - `DB2_USERNAME`
- Crias as seguintes integrações
- `GetDB2Token`
```json
{
  "openapi": "3.0.3",
  "info": {
    "title": "DB API Authentication",
    "version": "1.0.0",
    "description": "Endpoint para autenticação e geração de token."
  },
  "servers": [
        {
            "url": "https://{HOSTNAME}",
            "variables": {
                "HOSTNAME": {
                    "default": "example.db2.cloud.ibm.com"
                }
            }
        }
  ],
  "paths": {
    "/dbapi/v4/auth/tokens": {
      "post": {
        "summary": "Generate authentication token",
        "operationId": "generateAuthToken",
        "parameters": [
          {
            "name": "x-deployment-id",
            "in": "header",
            "required": true,
            "schema": {
              "type": "string"
            },
            "description": "Deployment identifier"
          }
        ],
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "required": [
                  "userid",
                  "password",
                  "separator",
                  "stop_on_error"
                ],
                "properties": {
                  "userid": {
                    "type": "string",
                    "example": "user123"
                  },
                  "password": {
                    "type": "string",
                    "example": "mypassword"
                  },
                  "separator": {
                    "type": "string",
                    "enum": [";"]
                  },
                  "stop_on_error": {
                    "type": "string",
                    "enum": ["no"]
                  }
                }
              }
            }
          }
        },
        "responses": {
          "200": {
            "description": "Authentication token generated",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "userid": {
                      "type": "string",
                      "example": "user123"
                    },
                    "token": {
                      "type": "string",
                      "example": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```
- `SendDb2Query`
```json
{
    "openapi": "3.0.3",
    "info": {
        "title": "Db2 SQL Jobs API",
        "version": "1.0.0",
        "description": "API para submissão de comandos SQL assíncronos no Db2 on Cloud."
    },
    "servers": [
        {
            "url": "https://{HOSTNAME}",
            "variables": {
                "HOSTNAME": {
                    "default": "example.db2.cloud.ibm.com"
                }
            }
        }
    ],
    "paths": {
        "/dbapi/v4/sql_jobs": {
            "post": {
                "summary": "Submeter Job SQL",
                "operationId": "submitSqlJob",
                "parameters": [
                    {
                        "name": "Authorization",
                        "in": "header",
                        "required": true,
                        "description": "Token Bearer de autenticação.",
                        "schema": {
                            "type": "string",
                            "example": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
                        }
                    },
                    {
                        "name": "x-deployment-id",
                        "in": "header",
                        "required": true,
                        "description": "Identificador do deployment do serviço.",
                        "schema": {
                            "type": "string",
                            "example": "zzz"
                        }
                    }
                ],
                "requestBody": {
                    "required": true,
                    "content": {
                        "application/json": {
                            "schema": {
                                "type": "object",
                                "required": [
                                    "commands"
                                ],
                                "properties": {
                                    "commands": {
                                        "type": "string",
                                        "example": "select * FROM DISCIPLINAS"
                                    },
                                    "limit": {
                                        "type": "integer",
                                        "enum": [10]
                                    },
                                    "separator": {
                                        "type": "string",
                                        "enum": [";"]
                                    },
                                    "stop_on_error": {
                                        "type": "string",
                                        "enum": [
                                            "no"
                                        ]
                                    }
                                }
                            }
                        }
                    }
                },
                "responses": {
                    "200": {
                        "description": "Job criado com sucesso",
                        "content": {
                            "application/json": {
                                "schema": {
                                    "type": "object",
                                    "properties": {
                                        "id": {
                                            "type": "string",
                                            "example": "1234567890abcdef"
                                        }
                                    }
                                }
                            }
                        }
                    },
                    "400": {
                        "description": "Requisição inválida"
                    },
                    "401": {
                        "description": "Não autorizado"
                    }
                }
            }
        }
    }
}
```
- `GetDb2QueryResult`
```json
{
    "openapi": "3.0.3",
    "info": {
        "title": "Db2 SQL Job Result API",
        "version": "1.0.0",
        "description": "API para consultar o resultado de um SQL Job no Db2 on Cloud."
    },
    "servers": [
        {
            "url": "https://{HOSTNAME}",
            "variables": {
                "HOSTNAME": {
                    "default": "example.db2.cloud.ibm.com"
                }
            }
        }
    ],
    "paths": {
        "/dbapi/v4/sql_jobs/{id}": {
            "get": {
                "summary": "Consultar resultado do SQL Job",
                "operationId": "getSqlJobResult",
                "parameters": [
                    {
                        "name": "id",
                        "in": "path",
                        "required": true,
                        "description": "Identificador do job SQL.",
                        "schema": {
                            "type": "string",
                            "example": "1772644599186_733471460"
                        }
                    },
                    {
                        "name": "Authorization",
                        "in": "header",
                        "required": true,
                        "description": "Token Bearer de autenticação.",
                        "schema": {
                            "type": "string",
                            "example": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
                        }
                    },
                    {
                        "name": "x-deployment-id",
                        "in": "header",
                        "required": true,
                        "description": "Identificador do deployment do serviço.",
                        "schema": {
                            "type": "string",
                            "example": "zzz"
                        }
                    }
                ],
                "responses": {
                    "200": {
                        "description": "Resultado do SQL Job",
                        "content": {
                            "application/json": {
                                "schema": {
                                    "type": "object",
                                    "properties": {
                                        "id": {
                                            "type": "string"
                                        },
                                        "status": {
                                            "type": "string",
                                            "example": "completed"
                                        },
                                        "results": {
                                            "type": "array",
                                            "items": {
                                                "type": "object",
                                                "properties": {
                                                    "rows_count": {
                                                        "type": "integer",
                                                        "description": "Quantidade de linhas retornadas.",
                                                        "example": 32
                                                    },
                                                    "rows": {
                                                        "type": "array",
                                                        "description": "Matriz contendo os resultados da consulta.",
                                                        "items": {
                                                            "type": "array",
                                                            "items": {
                                                                "type": "string"
                                                            }
                                                        }
                                                    }
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    },
                    "400": {
                        "description": "Requisição inválida"
                    },
                    "401": {
                        "description": "Não autorizado"
                    },
                    "404": {
                        "description": "Job não encontrado"
                    }
                }
            }
        }
    }
}
```
