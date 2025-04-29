# Chatroom Terraform Deployment

This repository contains Terraform configurations to deploy a full-stack chatroom application on AWS, including:

- **Go backend** on ECS Fargate behind an Application Load Balancer (ALB)
- **ECR** repository to store the backend Docker image
- **React frontend** hosted in a private S3 bucket and served via CloudFront with proper routing for API and WebSocket calls
- **CloudWatch Logs** for backend logging

## Prerequisites

- **AWS CLI** installed and configured with a profile that has permissions for ECS, ECR, S3, CloudFront, ALB, and CloudWatch
- **Terraform** v1.2.0 or newer
- **Docker** installed
- **Node.js** and **npm** (for building the React frontend)

## Quickstart

1. **Clone this repo**

2. **Deploy infrastructure**
   ```bash
   terraform init
   terraform apply 
   ```

3. **Build & push the backend Docker image**
   ```bash
   aws ecr get-login-password --region $REGION \
     | docker login --username AWS --password-stdin $REPO_URL

   docker build -t chatroom-backend:latest ../chatroom-go
   docker tag chatroom-backend:latest $REPO_URL:latest

   docker push $REPO_URL:latest
   ```

4. **Build & sync the frontend**
   ```bash
   cd ../chatroom-react
   npm install
   npm run build

   aws s3 sync dist s3://$BUCKET --region $REGION
   ```

Below I have also added some documentation and a how-to-install in Finnish for coursework.

# Terraform

Terraform on Hashicorpin kehittämä Infra as Code ratkaisu, joka mm. auttaa huomattavasti
automatisoimaan AWS:ään palvelimen pystytystä. Sen käyttö kuitenkin vaatii AWS-ympäristön
sekä CLI:n ymmärtämistä.

Terraformilla on tähän projektiin tehty ns. proof-of-concept, jolla saatiin yllä olevista
containereista backend-osio siirrettyä Elastic Container Registryn käytettäväksi, jolla
Elastic Container Service voi skaalata tarvittaessa lisää kontteja pystyyn. Frontend 
on tässä pystytetty huomattavasti halvemmalla ratkaisulla, S3:een. Pyyntöjen reitittäminen
on delegoitu Cloudfrontille, joka tulee halvemmaksi verrattuna kontitettuun reitittäjään.

## 1) AWS CLI käyttökuntoon - Käyttöoikeudet

Jotta Terraformin koodin saa ajettua onnistuneesti tarvitaan ensimmäisenä AWS-käyttäjä, 
jolla on riittävät oikeudet. Suosittelen itse menemään principle of least privileges 
reittiä. Tällöin, jos käyttäjän tiedot vuotavat jonnekin session aikana, voidaan tuhon
määrä pyrkiä minimoimaan. 

IAM Identity Centerissä tulee luoda uusi permission setti, jossa on seuraavat AWS-managed 
policyt:
```
AmazonEC2ContainerRegistryFullAccess
AmazonECS_FullAccess
AmazonS3FullAccess
AutoScalingFullAccess
CloudWatchFullAccessV2
ElasticLoadBalancingFullAccess
```

Lisäksi laitettu yksi Inline policy
```yaml
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": [
                "arn:aws:iam::*:role/ecsTaskExecutionRole",
                "arn:aws:iam::*:role/aws-service-role/ecs.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling"
            ]
        }
    ]
}
```

Sen jälkeen luodaan uusi käyttäjä IAM Identity Center -> Users -> Add user. Käyttäjän 
luonnin jälkeen IAM Identity Center -> AWS Accounts, mennään oman root käyttäjän luo 
painetaan Assign users or groups, liitetään juuri luotu uusi käyttäjä ja permission 
set yhteen. Uudelle käyttäjälle tulee myös tehdä MFA. Kaiken tämän jälkeen käyttäjää 
voi alkaa käyttämään AWS CLI:llä. Käydään hakemassa AWS access portal URL IAM IC:n 
dashboardista.

## 2) AWS CLI käyttökuntoon - Sessionin aloitus

Kirjaudutaan sisään terminaalissa 
```bash
$ aws configure sso
SSO session name (Recommended): esimerkkisession
SSO start URL [None]: # AWS access portal URL tänne
SSO region [None]: eu-central-1 # tai mikä vaan region
SSO registration scopes [sso:account:access]: # registration scope defaulttina riittää
Attempting to automatically open the SSO authorization page in your default browser.
```

Tämän jälkeen kirjaudutaan selaimessa, jonka jälkeen jatkuu terminaalissa. Hakasuluissa 
oleva teksti on, mitä täyttyy oletuksena jos antaa tyhjän rivin. Useampaan näistä voi 
antaa oletusvastauksen.
```bash
There are 2 AWS accounts available to you.
SSO account ID [None]: 123456789012
There are 1 roles available to you.
  1) permissionSetName
SSO role name [None]: permissionSetName
CLI default client Region [eu-central-1]:
CLI default output format [json]:
Which CLI profile should these credentials be stored under?
CLI profile name [esimerkkisession]:
```

## 3) Terraformin alustus

Nyt kun AWS CLI:in on kirjauduttu sisään oikeilla oikeuksilla, voidaan ajaa Terraform
koodilla infrat pystyyn. Terraformin asennukseen ohjeet löytyvät täältä (https://developer.hashicorp.com/terraform/install).

Testiä varten tehdään uusi hakemisto "terraforming".
```bash
$ mkdir terraforming
$ cd terraforming
$ git clone https://github.com/p-lemonish/ecs-chatroom
$ cd ecs-chatroom 
$ terraform init
$ terraform apply
```

HUOM. tarvittaessa saattaa joutua muuttamaan `variables.tf` sisällöstä `bucket-name` 
uniikiksi. Ongelman huomaa kyllä, kun terraform ei voi luoda S3 buckettia.

Kun terraformin kyselyyn vastaa "yes", alkaa se ajamaan koodissa määritettyä infraa 
pystyyn.

Vielä viimeistä vaihetta varten tarvitaan muutama tieto AWS:ltä. Jotta dockerin kontti
saadaan siirrettyä ECR:lle tarvitaan sen repositoryn osoite.

## 3) ECR-repo-URL:n hakeminen

Terraform on luonut ECR-repositorioksi oletuksena `chatroom-tf-backend`. Haetaan sen URI:

```bash
export AWS_PROFILE=permissionSetName
export AWS_REGION=eu-central-1

ECR_URI=$(aws ecr describe-repositories \
  --repository-names chatroom-tf-backend \
  --query "repositories[0].repositoryUri" \
  --output text \
  --region $AWS_REGION \
  --profile $AWS_PROFILE)
```

## 4) Go-backendin rakentaminen ja pushaus ECR:ään

```bash
git clone https://github.com/p-lemonish/chatroom-go
cd chatroom-go

docker build -t chatroom-backend .
docker tag chatroom-backend:latest $ECR_URI:latest

aws ecr get-login-password \
  --region $AWS_REGION \
  --profile $AWS_PROFILE \
| docker login \
  --username AWS \
  --password-stdin ${ECR_URI%/*}

docker push $ECR_URI:latest

cd ..
```

## 5) React-frontend rakentaminen ja synkronointi S3:een

```bash
git clone https://github.com/p-lemonish/chatroom-react
cd chatroom-react

npm install
npm run build

aws s3 sync ./dist \
  s3://chatroom-bucket-tf \
  --region $AWS_REGION \
  --profile $AWS_PROFILE

cd ..
```

Ja nyt Cloudfrontin takana oleva URL (löytyy esim. consolesta) tulisi näyttää 
chathuoneen sisällön ja sen tulisi myös toimia.
