# Aplicação em Alta Disponibilidade na AWS

**(INTRODUÇÃO)** Antigamente se utilizava uma máquina virtual na Oracle VM para hospedar a aplicação ou até servidores físicos, já hoje em dia se utiliza a AWS, o cenário mais básico
é a aplicação e o banco de dados rodando em uma EC2 ou no RDS (serviço de banco de dados relacional da AWS), mas essa arquitetura não é em alta
disponibilidade, tem outros métodos para se chegar na alta disponibilidade que será demonstrado ao decorrer do README.

**Primeiramente para entender o conceito desde o ínicio, farei a aplicação (chamada BIA) de forma simples na AWS para entender como isso aqui funciona:**

![image](https://github.com/DHiramIII/aws-alta-dispo/assets/74575487/b6bc2966-37dd-4cb2-b28a-6934a984c51c)

essa é como a aplicação fica ela será "rodada" em um container dentro de uma EC2 e o BD (postgresql) dentro da Instância também por enquanto.

# Criação da Role para acesso e demais usabilidades

A criação da role é importante para que as permissões sejam dadas para a Instância conseguir realizar as suas atividades ela pode ser feita tanto pela console e pela AWS CloudShell segue o script que será
executado como .sh:
```bash
role_name="role-acesso-ssm"
policy_name="AmazonSSMManagedInstanceCore"
    
if aws iam get-role --role-name "$role_name" &> /dev/null; then
    echo "A IAM role $role_name já existe."
    exit 1
fi
    
aws iam create-role --role-name $role_name --assume-role-policy-document file://ec2_principal.json
# Depois só trocar o nome desse ec2_principal.json por outro nome que vc desejar o importante é que o formato seja .json
    
# Cria o perfil de instância
aws iam create-instance-profile --instance-profile-name $role_name
    
# Adiciona a função IAM ao perfil de instância
aws iam add-role-to-instance-profile --instance-profile-name $role_name --role-name $role_name
    
aws iam attach-role-policy --role-name $role_name --policy-arn arn:aws:iam::aws:policy/$policy_name

# para adicionar uma nova policy é só criar mais uma variável e colocar o comando de cima

    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }]
    }
```
este é o script em json da role o ideal é criar um arquivo dentro do cloud shell para esse do json em formato json e o da role em .sh
porém apenas a role "AmazonSSMManagedInstanceCore" não será o suficiente também deve ser colocada as que estão na imagem abaixo:

![image](https://github.com/DHiramIII/aws-alta-dispo/assets/74575487/9ebc6164-16ba-42a6-973c-873c350da50a)

# Criação do Security Group

Esse passo pode tanto ser feito no cloudshell ou na console segue o script para criar um SG com as seguintes regras de entrada, 
coloque esse comando em um arquivo .sh e dê as permissões:

```bash
#!/bin/bash
    
# Variaveis
NOME="bia-dev"
Descricao="acesso ao bia-dev"
VPC="SEU VPC ID AQUI"
    
# Criando o SG
ID=$(aws ec2 create-security-group --group-name $NOME --description "$Descricao" --vpc-id $VPC --query 'GroupId' --output text)
    
# Adicionando Inbound Rules
    
aws ec2 authorize-security-group-ingress --group-id sg-0123456789abcdef0 --protocol tcp --port 3001 --cidr 0.0.0.0/0

    
echo "Security Group Created"
```

# Criação da Instância

Subir uma instâcia na AWS pode ser feito tanto pelo AWS CloudShell ou pela console o script para criar a instância dessa aplicação em específico é esse:

```bash
vpc_id=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query "Vpcs[0].VpcId" --output text)
subnet_id=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$vpc_id Name=availabilityZone,Values=us-east-1a --query "Subnets[0].SubnetId" --output text)
security_group_id=$(aws ec2 describe-security-groups --group-names "bia-dev" --query "SecurityGroups[0].GroupId" --output text 2>/dev/null)
    
if [ -z "$security_group_id" ]; then
    echo ">[ERRO] Security group bia-dev não foi criado na VPC $vpc_id"
    exit 1
fi
    
aws ec2 run-instances --image-id ami-02f3f602d23f1659d --count 1 --instance-type t3.micro \
--security-group-ids $security_group_id --subnet-id $subnet_id --associate-public-ip-address \
--block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":15,"VolumeType":"gp2"}}]' \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=bia-dev}]' \
--iam-instance-profile Name=role-acesso-ssm --user-data file://user_data_ec2_zona_a.sh
```
CONTEÚDO DO ARQUIVO "user_data_ec2_zona_a.sh" (crie ele com o comando nano "nome-do-arquivo.sh" e salve no CloudShell

Fornece um script de inicialização (user_data) a ser executado quando a instância é iniciada. O conteúdo do script user_data_ec2_zona_a.sh será executado na inicialização da instância.

```bash
#!/bin/bash
    
#Instalar Docker e Git
sudo yum update -y
sudo yum install git -y
sudo yum install docker -y
sudo usermod -a -G docker ec2-user
sudo usermod -a -G docker ssm-user id ec2-user ssm-user
sudo newgrp docker
    
#Ativar docker
sudo systemctl enable docker.service
sudo systemctl start docker.service

#Instalar docker compose 2
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/download/v2.23.3/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
    
    
#Adicionar swap
sudo dd if=/dev/zero of=/swapfile bs=128M count=32
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo echo "/swapfile swap swap defaults 0 0" >> /etc/fstab
    
    
#Instalar node e npm
curl -fsSL https://rpm.nodesource.com/setup_21.x | sudo bash -
sudo yum install -y nodejs    
```    

é bom guardar pq pode ser utilizado de outra forma (só alterar algumas coisas) aqui ele já vai criar a instância colocar ela na VPC certa, no SG (SECURITY GROUP) certo e na subnet certa
e também ele já vai associar a role para conseguir acesso SSM.

# Docker e os Containers

No Docker, containers e imagens são dois conceitos fundamentais que facilitam o desenvolvimento, distribuição e execução de aplicações de maneira consistente e isolada. Aqui está uma explicação resumida de cada um:

**Imagens**

Definição: Uma imagem Docker é um pacote leve, autossuficiente e imutável que contém tudo o que é necessário para executar uma aplicação: código, runtime, bibliotecas, variáveis de ambiente e configurações.

Funcionamento: As imagens são construídas a partir de um arquivo chamado Dockerfile, que contém uma série de instruções para configurar o ambiente da aplicação.


Reutilização: Uma imagem pode ser usada para criar múltiplos containers. Imagens são armazenadas em repositórios, como o Docker Hub, onde podem ser compartilhadas e distribuídas.

**Containers**

Definição: Um container é uma instância executável de uma imagem Docker. Ele encapsula a aplicação e suas dependências em um ambiente isolado, garantindo que ela rode de maneira consistente, independentemente do ambiente em que está sendo executada.

Funcionamento: Containers são iniciados a partir de imagens e podem ser parados, reiniciados e destruídos sem afetar o sistema host. Eles compartilham o kernel do sistema operacional host, mas operam de maneira isolada.

Escalabilidade: Containers são leves e rápidos de iniciar, o que os torna ideais para escalabilidade e deployment em larga escala.

**Resumo**

Imagens: Templates ou moldes para containers; contêm tudo que a aplicação precisa para rodar.

Containers: Instâncias em execução das imagens; executam a aplicação em um ambiente isolado e consistente.

# DockerFile

Normalmente já existem imagens criadas como a do Nginx a do Apache e também do Ubuntu entre várias outras porém algumas imagens, na nossa imagem em específico iremos utilizar o DockerFile para realizar o build dela, para cada aplicação será uma imagem, no caso dessa aqui o código do DockerFile (é bom que esse arquivo esteja na máquina, pode se trazer ele através do Github ou criar manualmente via nano) será o seguinte:

```bash
FROM node:21-slim
    
RUN npm install -g npm@latest --loglevel=error
WORKDIR /usr/src/app
    
COPY package*.json ./
    
RUN npm install --loglevel=error
    
COPY . .
    
RUN REACT_APP_API_URL=http://34.239.240.133 SKIP_PREFLIGHT_CHECK=true npm run build --prefix client
    
RUN mv client/build build
    
RUN rm  -rf client/*
    
RUN mv build client/
    
EXPOSE 8080
    
CMD [ "npm", "start" ]   
```
# Docker Compose

Docker Compose é uma ferramenta usada para definir e gerenciar aplicações Docker multi-containers. Basicamente, ele permite que você defina a configuração dos seus contêineres em um arquivo YAML (docker-compose.yml), onde você especifica os serviços que sua aplicação precisa (como bancos de dados, cache, etc.), suas dependências, volumes e redes.

**Componentes Principais do Docker Compose:**

Arquivo docker-compose.yml: É onde você define todos os serviços e suas configurações. Por exemplo, você pode definir um serviço web, um banco de dados e um cache Redis.

**Comandos do Docker Compose:**

`docker-compose up`: Inicializa e executa todos os contêineres definidos no arquivo.

`docker-compose down`: Para e remove todos os contêineres, redes e volumes criados pelo docker-compose up.

`docker-compose logs`: Exibe os logs dos contêineres.

`docker-compose build`: O docker-compose build procura as seções build no arquivo docker-compose.yml.
Para cada serviço que tem uma instrução build, ele executa o Dockerfile associado.
Se as imagens já existirem e não houver mudanças, ele pode reutilizar camadas de imagem existentes para acelerar o processo.

Outros comandos incluem `docker-compose start`, `docker-compose stop`, etc.

o arquivo `docker-compose.yml` será o seguinte:

```bash
version: "3"
services:
  server:
    build: .
    container_name: bia
    ports:
      - 3001:8080
    links:
      - database
    environment:
      DB_USER: postgres
      DB_PWD: postgres
      DB_HOST: database
      DB_PORT: 5432     
  database:
    image: postgres:16.1
    container_name: database
    environment:
      - "POSTGRES_USER=postgres"
      - "POSTGRES_PASSWORD=postgres"
      - "POSTGRES_DB=bia"
    ports:
      - 5433:5432
    volumes:
      - db:/var/lib/postgresql/data
volumes:
  db:
```

Em resumo, esse arquivo docker-compose.yml define um ambiente onde um servidor (aplicação) e um banco de dados PostgreSQL trabalham juntos, com a configuração necessária para se comunicarem e armazenarem dados de forma persistente (deve ser criado um arquivo nano ou tambem o arquivo docker-compose.yml pode ser "clonado" de algum repositório de Github).

LEMBRAR DEPOIS DO ERRO QUE DEU PQ NAO TINHA TODOS OS ARQUIVOS, PARA A APLICAÇÃO FUNCIONAR TEM QUE TER O FRONT-END O BACK-END TUDO DENTRO DA MÁQUINA PARA QUE FUNCIONE E ELE DEVE SER APONTADO NO DOCKERFILE

Depois daqui é só dar os comandos `docker compose up -d` que ele irá começar a buildar a aplicação.

**Após isso a aplicação estará no ar só pegar o ip público e acrescentar :3001 na barra de pesquisa que terá acesso, e dar o seguinte comando `docker compose exec server bash -c 'npx sequelize db:migrate'`









