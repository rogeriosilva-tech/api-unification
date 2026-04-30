# api-unification

![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Terraform](https://img.shields.io/badge/terraform-%235835CC.svg?style=for-the-badge&logo=terraform&logoColor=white)
![Go](https://img.shields.io/badge/go-%2300ADD8.svg?style=for-the-badge&logo=go&logoColor=white)
![NodeJS](https://img.shields.io/badge/node.js-6DA55F?style=for-the-badge&logo=node.js&logoColor=white)

Focado especificamente na configuração do serviço API Gateway

Para implementar a criação de endpoints com novo path (/api/v2) sem autenticação será implantada a solução técnica em três frentes: 
* configurar o API Gateway (HTTP API)
* criar as regras no ALB
* Garantir que os cabeçalhos de segurança sejam injetados.

### 1. Criar Integração e Rota no API Gateway (HTTP API)
Primeiro, você deve criar a integração que aponta para o Load Balancer desejado e em seguida, a rota específica /v2.

* Criar a Integração (VPC Link):

```bash
aws apigatewayv2 create-integration \
    --api-id <SEU_API_ID> \
    --integration-type HTTP_PROXY \
    --integration-method ANY \
    --connection-type VPC_LINK \
    --connection-id <SEU_VPC_LINK_ID> \
    --integration-uri <ARN_DO_LISTENER_DO_ALB> \
    --payload-format-version 1.0 \
    --request-parameters '{"append:header.X-Origin-Verify": "SEGREDO_COMPARTILHADO_PROD"}'
```
**Nota:** O parâmetro --request-parameters injeta o header de segurança que o backend validará.

* Criar a Rota:

```bash
aws apigatewayv2 create-route \
    --api-id <SEU_API_ID> \
    --route-key "ANY /hooks/api/v2/{proxy+}" \
    --target "integrations/<ID_DA_INTEGRACAO_CRIADA_ACIMA>" \
    --authorization-type "CUSTOM" \
    --authorizer-id <ID_DO_LAMBDA_AUTH>
```
### 2. Configurar Regras de Listener no ALB
Agora, no Load Balancer, precisamos garantir que o tráfego da /v2 só seja aceito se contiver o header correto.

### Criar regra de prioridade para a V2:
```bash
aws elbv2 create-rule \
    --listener-arn <ARN_DO_LISTENER_HTTPS_443> \
    --priority 10 \
    --conditions '[
        {
            "Field": "path-pattern",
            "Values": ["/hooks/api/v2/*"]
        },
        {
            "Field": "http-header",
            "HttpHeaderConfig": {
                "HttpHeaderName": "X-Origin-Verify",
                "Values": ["SEGREDO_COMPARTILHADO_PROD"]
            }
        }
    ]' \
    --actions Type=forward,TargetGroupArn=<ARN_DO_TARGET_GROUP_HOOKS>
```
### Criar regra de bloqueio (Segurança Extra):
Caso alguém tente acessar a /v2 diretamente pela URL pública do ALB sem o header, o ALB deve retornar 403.
```bash
aws elbv2 create-rule \
    --listener-arn <ARN_DO_LISTENER_HTTPS_443> \
    --priority 11 \
    --conditions '[
        {
            "Field": "path-pattern",
            "Values": ["/hooks/api/v2/*"]
        }
    ]' \
    --actions Type=fixed-response,FixedResponseConfig='{
        "MessageBody":"Acesso negado: Origem nao autorizada.",
        "StatusCode":"403",
        "ContentType":"text/plain"
    }'
```
### 3. Ajustes de Ambiente (Homolog/Sandbox)
   A solução deve ser aplicada a todos os ambientes, repetir o processo alterando as variáveis. Recomendação usar o parâmetro --profile ou exportar as variáveis de ambiente antes de executar:
```bash
# Exemplo para Homologação
export API_ID="id-homolog"
export SHARED_SECRET="segredo-homolog-2026"

# Execute os comandos acima usando as variáveis
```
### Checklist de Validação via CLI
Após executar, você pode validar se as rotas foram criadas corretamente:
```bash
Listar rotas do Gateway: aws apigatewayv2 get-routes --api-id <ID>
```
```bash
Verificar regras do ALB: aws elbv2 describe-rules --listener-arn <ARN>
```

## Fluxo de comunicação

Essa é a **chave** que transforma o  Load Balancer em um porteiro inteligente. Sem essa conexão, a rota /api/v2 ficaria exposta na internet para qualquer um acessar diretamente, ignorando o API Gateway e o Lambda Authorizer.

A conexão funciona como um aperto de mão secreto (Shared Secret) entre duas camadas de infraestrutura. Aqui está o passo a passo do fluxo:

### 1. O Carimbo no Gateway (Injeção)
Quando o frontend faz uma chamada para [apigw.nuvidio.com/hooks/api/v2/](https://apigw.nuvidio.com/hooks/api/v2/)..., a requisição passa primeiro pelo seu Lambda Authorizer. Se o token for válido, o API Gateway prossegue com a requisição.

Nesse momento, o Gateway é configurado para "injetar" um cabeçalho HTTP customizado (ex: X-Origin-Verify: SEGREDO_123) antes de encaminhar o tráfego para o seu ALB. Esse cabeçalho não existia na requisição original do usuário; ele foi adicionado por uma entidade de confiança (o seu Gateway).

### 2. O "Filtro" no ALB (Regra de Listener)
O Application Load Balancer recebe tráfego de dois lugares: da internet pública e do API Gateway. Para proteger a nova rota /v2, configuramos uma Listener Rule com duas condições obrigatórias:

**Condição A:** O caminho deve ser /api/v2/*.

**Condição B:** O cabeçalho X-Origin-Verify deve ser EXATAMENTE SEGREDO_123.

Se alguém tentar acessar a URL do ALB diretamente ([api.nuvidio.com/hooks/api/v2/](https://api.nuvidio.com/hooks/api/v2/)...), essa pessoa não terá o cabeçalho secreto. O ALB verá que a Condição B falhou e seguindo a regra criada, retornará um **403 Forbidden** ou passará para a próxima regra que exige Basic Auth.

### 3. O "Bypass no Backend
Dentro do código da Hooks API (Node) ou Reports API (Go), o middleware de segurança agora tem uma lógica simples:

"Se o tráfego vem para o path /v2, eu confio que o ALB já validou o **aperto de mão** e não exijo Basic Auth."

### Por que isso é seguro?
O segredo é interno: O usuário final nunca vê o valor do X-Origin-Verify. Ele é injetado atrás das cortinas entre o Gateway e o ALB.

**Centralização:** Só é necessário validar o JWT do usuário uma vez (no Lambda Auth do Gateway). O resto da infraestrutura apenas valida se a mensagem veio de uma fonte confiável.

**Proteção contra invasão direta:** Se um atacante descobrir o IP ou DNS do ALB, ele não conseguirá consumir a API sem o cabeçalho que só o Gateway possui.

### Resumo do Fluxo de Confiança:
* Usuário → Envia JWT → Gateway.

* Gateway → Valida JWT → Injeta Header Secreto → ALB.

* ALB → Checa Header Secreto → Encaminha → ECS (Aplicação).

* ECS → Vê o path /v2 → Processa a requisição sem pedir senha.

Isso fecha o ciclo da Opção A com segurança perimetral completa.
