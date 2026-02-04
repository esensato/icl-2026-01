### Acesso IBM Cloud

- Cadastro `IBM Academic Initiative`

    [IBM Academic Initiative](https://github.com/academic-initiative/documentation/blob/main/academic-initiative/how-to/How-to-register-with-the-IBM-Academic-Initiative/readme.md)

- Obter promocode para a `IBM Cloud`

    [IBM Cloud Promocode](https://github.com/academic-initiative/documentation/blob/main/academic-initiative/how-to/How-to-request-and-IBM-Cloud-Feature-Code/readme.md)

- Ativar `IBM Cloud`

    [IBM Cloud](https://github.com/academic-initiative/documentation/blob/main/academic-initiative/how-to/How-to-create-an-IBM-Cloud-account/readme.md)

### Language Translator

- Acesso via `postman`
    - Utilizar a URL `https://api.us-south.language-translator.watson.cloud.ibm.com/instances/INSTANCE_ID/v3/translate?version=2018-05-01`
    - Autenticação deve ser `Basic Auth` onde `Username` deve ser `apikey` e `Password` a chave de API
    
### Uso NLU

- Acesso via `postman`
    - Utilizar a URL `https://api.us-south.natural-language-understanding.watson.cloud.ibm.com/instances/INSTANCE_ID/v1/analyze?version=2019-07-12`
    - Autenticação deve ser `Basic Auth` onde `Username` deve ser `apikey` e `Password` a chave de API

- Biblioteca `nodejs`
    
    [Bliblioteca Nodejs](https://www.npmjs.com/package/ibm-watson)

- Instalar a biblioteca

    ```nodejs
    npm init -y
    npm i --save ibm-watson
    ```
- Categorias

```json
    {
        url: 'https://www.espm.br/',
        features: {
            categories: {}
        }
    }
```
- Classificações

```json
    {
        text: 'I feel good today!',
        features: {
            classifications: { model: 'tone-classifications-en-v1' }
        }
    }
```

- Somente disponível para textos em inglês

    - excited: Mostrando entusiasmo e interesse pessoais
    - frustrated: Sentindo-se incomodado e irritado
    - impolite: Sendo desrespeitoso e rude
    - polite: Exibindo um comportamento racional e orientado a objetivos
    - sad: Uma emoção passiva desagradável
    - satisfied: Uma resposta afetiva à qualidade do serviço percebida
    - sympathetic: Um modo afetivo de compreensão que envolve ressonância emocional

- Referêcia: [Análise de Tons](https://cloud.ibm.com/docs/natural-language-understanding?topic=natural-language-understanding-tone_analytics)

- Conceitos
```json
    {
        url: 'www.espm.br',
        features: {
            concepts: {}
        }
    }
```
- Entidades

```json
    {
        text: 'Eu quero que o meu pedido entregue na cidade de São Paulo, na Rua Antonio da Silva, 123',
        features: {
            entities: {}
        }

    }
```

### Watson Assistant
- Permite a criação de **chatbots**
#### Chatbot Secretaria Universidade
- Efetuar login na [IBM Cloud](https://cloud.ibm.com)
- Instanciar o serviço [Watson Assistant](https://cloud.ibm.com/catalog/services/watsonx-assistant)
- Utilizar o [ChatGPT](https://chatgpt.com/) para gerar os textos
- Criar uma `Persona` para o chatbot, por exemplo:
    ```dotnetcli
    Crie uma persona para um chatbot que auxilie alunos universitários da faculdade "Belo Diploma" 
    nas questões como: obter nota, faltas, grade de aulas, etc... Essa persona deve ter um ótimo 
    senso de humor e uma linguagem descontraída
    ```
- Criar o diálogo introdutório, o `On boarding`
- Adicionar 3 variações de resposta para quando a pergunta não for compreendida pelo Chatbot (escolhidas aleatoriamente)
    - Observar o `No matches count <= 3`;
- Criar a primeira ação: "Verificar as disciplinas matriculadas";
    - Aluno deve infomar o número de matrícula;
    - O diálogo deve informar "Estou pesquisando sua matrícula `numero_da_matricula`";
    - Criar uma variável de sessão para armazenar o número de matrícula do aluno;
    - Ao final, informar a grade de disciplinas e o nome do aluno (sempre o mesmo)
- Variáveis de sessão:

    <div style="width:100px; height:100px">
    <img src="img/img2.png">
    </div>

#### Integração com o sistema da universidade

<div style="width:100px; height:100px">
<img src="img/img3.png">
</div>

- Acessar o `endpoint` para obter a lista de disciplinas

    `https://sistema-universitario.glitch.me/grade/1000`

- Formato `OpenAPI`
```
Gere um json no formato OpenAPI para o endpoint https://sistema-universitario.glitch.me/grade/:matricula onde :matricula corresponde à matrícula do aluno. O endpoint retorna um JSON no formato {aluno: "nome do aulo", disciplinas: ["disciplina1", "disciplina2"]}
```

<div style="width:100px; height:100px">
<img src="img/img4.png">
</div>

- Desenvolver um diálogo para que o aluno possa solicitar a nota de uma disciplina informando o número de matrícula e a descrição da disciplina

    `https://sistema-universitario.glitch.me/nota/1000/Estrutura de Dados`

- Desenvolver um diálogo para que o aluno possa consultar a sala de aula de uma disciplina

    `https://sistema-universitario.glitch.me/sala/Estrutura de Dados`

### Personalizando o chatbot
- Instalar em uma página HTML

    [Instalar Assistente](https://developer.ibm.com/tutorials/embed-watson-assistant-in-website/)

- Código inicial para exibir o chatbot em um site
    ```javascript
    <script>
      window.watsonAssistantChatOptions = {
        // A UUID like '1d7e34d5-3952-4b86-90eb-7c7232b9b540' included in the embed code provided in IBM watsonx Assistant.
        integrationID: 'YOUR_INTEGRATION_ID',
        // Your assistants region e.g. 'us-south', 'us-east', 'jp-tok' 'au-syd', 'eu-gb', 'eu-de', etc.
        region: 'YOUR_REGION',
        // A UUID like '6435434b-b3e1-4f70-8eff-7149d43d938b' included in the embed code provided in IBM watsonx Assistant.
        serviceInstanceID: 'YOUR_SERVICE_INSTANCE_ID',
        // The callback function that is called after the widget instance has been created.
        onLoad: async (instance) => {
          await instance.render();
        }
      };
      setTimeout(function(){const t=document.createElement('script');t.src='https://web-chat.global.assistant.watson.appdomain.cloud/versions/' + (window.watsonAssistantChatOptions.clientVersion || 'latest') + '/WatsonAssistantChatEntry.js';document.head.appendChild(t);});
    </script>
    ```
- Para obter um exemplo, clicar em
    <div style="width:100px; height:100px">
    <img src="img/img1.png">
    </div>

- E em seguida, clicar na aba superior **Embed**
- `onLoad` executado quando o chatbot é carregado
- Configurações de *layout*
    ```json
        layout: {
            showFrame: true,
            hasContentMaxWidth: false,
        }
    ```
- Configurações do tema
    ```json
        themeConfig: {
            carbonTheme: 'g100',
            corners: 'round',
        }
    ```
- Obs: `carbonTheme` podem ser "white", "g10", "g90" ou "g100" e `corner` "square" ou "round"
- Botão para fechar o chatbot
    ```json
        headerConfig: {
            closeButtonIconType: 'side-panel-left',
        }
    ```
- Obs: opções "minimize", "close", "side-panel-left" e "side-panel-right".
#### Eventos
- Lista de eventos completa pode ser encontrada [Aqui](https://web-chat.global.assistant.watson.cloud.ibm.com/docs.html?to=api-events#event-list)
    ```javascript
        instance.on({
            type: 'receive', handler: (event) => { console.log('I received a message!', event); }
        });
    ```
- Evento `receive`: executado quando uma mensagem é recebida;
- - Os principais parâmetros recebidos pelas funçõs na varável `event` são:
    - `event.data`: mensagem (dados) recebidos pelo chatbot como respostas das intenções do usuário;
    - `event.data.output.generic`: itens da resposta recebidos (texto, etc...)
- Evento `pre:receive`: executado antes do `receive`;
    ```javascript
        instance.on({
            type: 'pre:receive', handler: (event) => {
                console.log('pre:receive')
                const message = event.data;
                if (message.output.generic) {
                    message.output.generic.forEach(messageItem => {
                        console.log(messageItem);
                        if (messageItem.response_type === 'text') {
                            messageItem.response_type = 'teste123';
                        }
                    })
                }
            }
        });
    ```

- Evento `customResponse`: permite criar uma resposta personalizada;
    ```javascript
        function customResponseHandler(event) {
            const { message, element, fullMessage } = event.data;

            const div = document.createElement('div');
            // obtem o texto da mensagem
            div.innerHTML = message.text;
            div.style.border = 'solid 1px';
            div.style.color = 'red';

            // message.options.forEach((messageItem, index) => {
            //     const button = document.createElement('button');
            //     button.innerHTML = messageItem.label;
            //     button.classList.add('CardButton');
            //     button.addEventListener('click', () => onClick(messageItem, button, fullMessage, index));
            //     element.appendChild(button);
            // });

            element.appendChild(div);

        }
    ```