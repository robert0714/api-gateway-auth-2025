## api-gateway-auth

**[Feature request](https://github.com/aws-samples/api-gateway-auth/issues/new)** | **[Detailed blog post](https://aws.amazon.com/blogs/compute/automating-mutual-tls-setup-for-amazon-api-gateway/)**

This sample application showcases how to set up and automate different types of authentication supported by 
[Amazon API Gateway HTTP API](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api.html) via [AWS SAM](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html)

- [Mutual TLS](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-mutual-tls.html)
- [JWT authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-jwt-authorizer.html)
- [AWS Lambda authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-lambda-authorizer.html)
- [IAM authorization](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-access-control-iam.html) (Not supported via SAM. Ref [issue](https://github.com/aws/aws-sam-cli/issues/2233))]

This SAM app uses java as language runtime for the lambda functions and custom resources.

# Setup 

## All authentications methods including mTLS:

The main SAM [template-all-auth.yaml](template-all-auth.yaml) is used to set up HTTP API and different types of auth mentioned above.
As a pre requisite step, in order to configure JWT authorizer, you will need to run [template-cognito.yaml](template-cognito.yaml)
to setup [Amazon Cognito](https://aws.amazon.com/cognito/) as the JWT token provider. If you wish to have and 
HTTP API setup with only mTLS, follow section [Only mTLS with HTTP API setup](#Only mTLS with HTTP API setup). Lets begin.

#### Setup JWT Token provider

This will end up creating cognito user pool which we will use to set up our HTTP API with different auths.
This is needed because we will use [Amazon Cognito](https://aws.amazon.com/cognito/) as the JWT token provider.
You can skip this step if you are not going to configure JWT Authorizer for your HTTP API 
in [template-all-auth.yaml](template-all-auth.yaml#L174)

```
    api-gateway-auth$ sam build -t template-cognito.yaml
```

```
    api-gateway-auth$ sam deploy -t .aws-sam/build/template.yaml -g

    Deploying with following values
    ===============================
    Stack name                 : jwt-auth
    Region                     : eu-west-1
    Confirm changeset          : False
    Deployment s3 bucket       : aws-sam-cli-managed-default-samclisourcebucket-randomhash
    Capabilities               : ["CAPABILITY_IAM"]
    Parameter overrides        : {'AppName': 'jwt-auth', 'ClientDomains': 'http://localhost:8080', 'AdminEmail': 'emailaddress@example.com', 'AddGroupsToScopes': 'true'}

```

#### Set up HTTP API

```
    api-gateway-auth$ sam build -t template-all-auth.yaml
```

```
    api-gateway-auth$ sam deploy -t .aws-sam/build/template.yaml

    Deploying with following values
    ===============================
    Stack name                 : http-api-authdemo
    Region                     : eu-west-1
    Confirm changeset          : False
    Deployment s3 bucket       : aws-sam-cli-managed-default-samclisourcebucket-randomhash
    Capabilities               : ["CAPABILITY_IAM"]
    Parameter overrides        : {'UserPoolId': 'from previous stack output', 'Audience': 'from previous stack output', 'HostedZoneId': 'Hosted zone id for custom domain', 'DomainName': 'domain name for the http api', 'TruststoreKey': 'truststore.pem'}
```

## Only mTLS with HTTP API setup

#### Set up HTTP API

```
    api-gateway-auth$ sam build
```
![sam build](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2020/12/17/mtls2.jpg)
```
    api-gateway-auth$ sam deploy -t

    Deploying with following values
    ===============================
    Stack name                 : http-api-authdemo
    Region                     : eu-west-1
    Confirm changeset          : False
    Deployment s3 bucket       : aws-sam-cli-managed-default-samclisourcebucket-randomhash
    Capabilities               : ["CAPABILITY_IAM"]
    Parameter overrides        : {'HostedZoneId': 'Hosted zone id for custom domain', 'DomainName': 'domain name for the http api', 'TruststoreKey': 'truststore.pem'}
```
![sam build](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2020/12/17/mtls3.jpg)

Or
```bash
aws cloudformation deploy \
    --template-file template.yaml \
    --stack-name http-api-authdemo \
    --region $REGION \
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
    --parameter-overrides \
        HostedZoneId=$HOSTED_ZONE_ID \
        DomainName=$DOMAIN_NAME \
        TruststoreKey=$TRUSTSTORE_KEY
```
Or
```bash
 sam deploy --s3-bucket $CF_BUCKET \
    --stack-name http-api-authdemo \
    --capabilities CAPABILITY_IAM
    --parameter-overrides \
        HostedZoneId=$HOSTED_ZONE_ID \
        DomainName=$DOMAIN_NAME \
        TruststoreKey=$TRUSTSTORE_KEY
```
## Testing and validation

At this point, your stack should update successfully and you will have a HTTP API with Mutual TLS setup by default using 
[AWS Certificate Manager Private Certificate Authority](https://aws.amazon.com/certificate-manager/private-certificate-authority/).

Stack will also generate one of the client certificates for you to validate the API and output its ARN as stack output

####Export certificate via Console:

- Navigate to [AWS Certificate Manager console](https://console.aws.amazon.com/acm/home). You will find a private 
certificate already generated with Name as ClientOneCert.

- Select the cert and under action choose `Export (Private certificates only)`. Enter passphrase on next screen which 
will be needed to decrypt the `Certificate private key` later.

- `Export certificate body to a file` and `Export certificate private key to a file`

#### Export certificate via CLI:

```
aws acm export-certificate --certificate-arn <<Certificat ARN from stack output>> --passphrase $(echo -n 'your paraphrase' | base64) --no-cli-auto-prompt --region eu-west-1 | jq -r '"\(.Certificate)"' > client.pem
```

```
aws acm export-certificate --certificate-arn <<Certificat ARN from stack output>> --passphrase $(echo -n 'your paraphrase' | base64) --no-cli-auto-prompt --region eu-west-1 | jq -r '"\(.PrivateKey)"' > client.encrypted.key
```

#### Decrypt the private key

- Decrypt private key downloaded using below command:

```
    openssl rsa -in <<Encrypted file>> -out client.decrypted.key

    Enter pass phrase for client.encrypted.txt:
    writing RSA key
```

#### Call the HTTP API to validate mTLS

- Now you should be able to access the configured api with different paths and auth methods using mutual TLS.

```
    curl -v --cert client.pem  --key client.decrypted.key https://<<api-auth-demo.domain.com>>
```

## Auth0 setup for REST and HTTP API

API gateway both REST and HTTP can be configured to work with [Auth0](https://auth0.com/). There is a sample template
[template-auth0.yaml](template-auth0.yaml) which sets up sample REST and HTTP Api to work with Auth0.

Template expects two parameters:

- IssuerUrl: The issuer of the token. Use https://YOUR_DOMAIN/. Be sure to include the trailing slash. 
- APIAudience: The identifier value of the API you created in the Auth0 API.

HTTP API will be set up using native [JWT Authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-jwt-authorizer.html)
while REST API will be set up using [Token based Lambda Authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html) 
to integrate with Auth0.

### Setup

```
    api-gateway-auth$ sam build -t template-auth0.yaml
```

```
    api-gateway-auth$ sam deploy -t .aws-sam/build/template.yaml
```

## Credits

Setting up of JWT authorizer is inspired from example in [sessions-with-aws-sam](https://github.com/aws-samples/sessions-with-aws-sam/tree/master/cognito)  

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

# SAM LOCAL
## Ready
```bash
vagrant up
vagrant ssh
sudo apt install -y wget unzip


curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

wget https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip
unzip aws-sam-cli-linux-x86_64.zip -d sam-installation
sudo ./sam-installation/install
sam --version
```
## Test in sam Local
```bash
sam local start-api
```
another terminal
```bash
vagrant@sam:~$ curl --location 'http://127.0.0.1:3000' 

vagrant@sam:~$ curl --location 'http://127.0.0.1:3000/simple'  
vagrant@sam:~$ curl --location 'http://127.0.0.1:3000/admin'  
vagrant@sam:~$ curl --location 'http://127.0.0.1:3000/service-role'  
```
```bash

$ sam deploy --s3-bucket $CF_BUCKET --stack-name api-gateway-auth --capabilities CAPABILITY_IAM
```
