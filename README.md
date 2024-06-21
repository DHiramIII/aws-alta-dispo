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

este é o script em json da role o ideal é criar um arquivo dentro do cloud shell para esse do json em formato json e o da role em .sh
porém apenas a role "AmazonSSMManagedInstanceCore" não será o suficiente também deve ser colocada as que estão na imagem abaixo:

![image](https://github.com/DHiramIII/aws-alta-dispo/assets/74575487/9ebc6164-16ba-42a6-973c-873c350da50a)

# Criação do Security Group

Esse passo pode tanto ser feito no cloudshell ou na console segue o script para criar um SG com as seguintes regras de entrada, 
coloque esse comando em um arquivo .sh e dê as permissões:

    #!/bin/bash
    
    # Variaveis
    NOME="bia-dev"
    Descricao="acesso ao bia-dev"
    VPC="vpc-0ff61e8578eee23a5"
    
    # Criando o SG
    ID=$(aws ec2 create-security-group --group-name $NOME --description "$Descricao" --vpc-id $VPC --query 'GroupId' --output text)
    
    # Adicionando Inbound Rules
    
    aws ec2 authorize-security-group-ingress --group-id sg-0123456789abcdef0 --protocol tcp --port 3001 --cidr 0.0.0.0/0

    
    echo "Security Group Created"

# Criação da Instância

Subir uma instâcia na AWS pode ser feito tanto pelo AWS CloudShell ou pela console o script para criar a instância dessa aplicação em específico é esse:


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

CONTEÚDO DO ARQUIVO "user_data_ec2_zona_a.sh" (crie ele com o comando nano "nome-do-arquivo.sh" e salve no CloudShell

Fornece um script de inicialização (user_data) a ser executado quando a instância é iniciada. O conteúdo do script user_data_ec2_zona_a.sh será executado na inicialização da instância.

    #!/bin/bash
    
    #Instalar Docker e Git
    sudo yum update -y
    sudo yum install git -y
    sudo yum install docker -y
    sudo usermod -a -G docker ec2-user
    sudo usermod -a -G docker ssm-user
    id ec2-user ssm-user
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
    

é bom guardar pq pode ser utilizado de outra forma (só alterar algumas coisas) aqui ele já vai criar a instância colocar ela na VPC certa, no SG (SECURITY GROUP) certo e na subnet certa
e também ele já vai associar a role para conseguir acesso SSM.



