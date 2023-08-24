Atividade de Docker
Esse é o repositório da atividade de Docker do programa de bolsas da CompassUOL. Aqui será dado o passo a passo para que qualquer um possa recriar esse projeto.

Obs: Os nomes de algumas ferramentas que envolvem os serviços da AWS serão dados em inglês, visto que a console AWS utilizada para a atividade estava configurada para a língua inglesa.

Sobre
Nessa atividade, o objetivo era rodar uma aplicação WordPress num container Docker em instâncias EC2 que estariam sendo escaladas pelo serviço de Auto Scaling da AWS. As instâncias deveriam poder rodar em AZs diferentes e, ao mesmo tempo, estarem também conectadas a um banco de dados MYSQL criado pelo serviço de RDS(Relational Database Service) da AWS. O tráfego gerado pelo acesso à aplicação seria balanceado por um Load Balancer. Na imagem abaixo está ilustrado a descrição feita anteriormente:

image

Configurações da AWS
Antes de mais nada, é preciso ter uma conta na AWS e, nela, ter acesso aos serviços de EC2, VPC, EFS e RDS. Eles serão essenciais para que tudo funcione corretamente.

Security Groups
Serão necessários 4 security groups para esse projeto: um para as instâncias, um segundo para o load balancer, um terceiro para o EFS e mais um para o RDS.

O primeiro deve liberar a porta 22(SSH) para a source "Meu IP" e a porta 80 para o security group do load balancer(ou seja, apenas será possível acessar a porta 80 por meio do load balancer):

image

O segundo liberará a porta 80 para todas as sources:

image

O terceiro liberará a porta 80 e a porta 2049(NFS/EFS) para todas as sources:

image

E o último liberará a porta 3306(MYSQL) para o security group das instâncias, permitindo que todas elas acessem o DB:

image

VPC
A VPC desse projeto vai contar com duas subnets privadas(cada uma em uma AZ) e um route table para cada subnet, ambos apontando para um NAT Gateway; E duas subnets públicas(cada uma em uma AZ também) com um route table apontando para um internet gateway. O resultado será semelhante ao da imagem:

image

image

Instância EC2
Por mais que o auto scaling vá cuidar da criação de instâncias futuramente, é preciso que criemos uma agora para a utilizarmos como template. Suas configurações devem ser as seguintes:

Tipo t3.small
8GB gp2 de volume
AMI Amazon Linux 2
Security Group e VPC criados anteriormente, selecionando uma subnet privada
Criar um key pair(chave pública e chave privada) ou utilizar um já existente
Tendo criado a instância, selecione-a, clique em Actions > Image e templates > Create template from instance e siga os passos:

image

Dê um nome e uma descrição para seu template
Se certifique de que as configurações de instância do template estão idênticas às citadas anteriormente
Em advanced details, copie e cole o conteúdo do script(disponível como user_data.sh nesse repositório ou mais abaixo nessa documentação) no user data ou simplesmente faça upload do arquivo
Clique em Create launch template
Com o template criado, poderemos utilizá-lo futuramente no nosso auto scaling.

Também seria interessante associar um Elastic IP à instância, pois um IP público seria necessário caso você queira fazer acesso via SSH posteriormente. Teoricamente, não seria preciso fazer qualquer acesso à instância nesse projeto, visto que o script fará todas as configurações por nós. Entretanto, caso você se veja numa situação em que um acesso se faz necessário, aí vai um passo a passo: caso seu PC seja um linux, basta abrir o Terminal e executar o seguinte comando, substituindo os nomes dados apenas como exemplo:

ssh -i privateKey.pem ec2-user@public-IPv4-address
No caso dos usuários de Windows, o ideal seria utilizar o software de acesso remoto chamado puTTY, juntamente com o gerador de chaves de acesso puTTYgen. Você pode instalar o puTTY aqui e o puTTYgen será instalado em conjunto. O uso do puTTYgen é necessário aqui pois, por mais que tenhamos uma chave privada do tipo .pem, que foi criada junto com a instância, o puTTY só aceita o uso de chaves .ppk, e assim o puTTYgen entra em cena para gerar uma chave .ppk a partir da nossa chave .pem.

Quando o puTTY for instalado com sucesso, abra o menu Iniciar, procure a pasta do puTTY na lista de apps, abra a pasta e clique no puTTYgen.

image

Com o puTTYgen aberto, se certifique que o tipo de chave que será gerado é RSA; feito isso, clique em Load, selecione a opção no canto inferior direito que exibe todos os arquivos, procure e selecione o arquivo .pem da chave privada gerada no processo de criação da instância EC2.

image

image

Agora, basta clicar em Save private key, dar um nome a chave nova(de preferência o mesmo nome, para evitar confusões) e salvá-la no destino que preferir.

Com a chave .ppk em mãos, abra o puTTY e siga os seguintes passos:

No painel Category, expanda Connection, SSH, Auth e clique em Credentials
Em Credentials, clique no botão Browse ao lado de Private key for authentication e selecione a chave .ppk criada anteriormente
image

No painel Category, clique em Session
Em Hostname(or IP address), insira os dados no formato abaixo:
ec2-user@DNS-IPv4-público
Se certifique de que o valor em Ports é 22 a opção selecionada em Connecetion type é SSH
(Opcional) Caso queira salvar essas informações para acessar a instância novamente no futuro mais facilmente, dê um nome a ela em Saves Sessions e clique em Save
Clique em Open. O puTTY exibirá uma caixa de diálogo por medidas de segurança no seu primeiro acesso dandos alguns alertas, mas basta clicar em Accept e você terá acesso a sua instância
Load Balancer e Target Group
Vamos começar essa etapa da atividade criando o Target Group. Para isso, selecione o serviço de EC2 na console AWS e procure por Target Groups. Em Target Groups, clique em Create Target Group e siga os seguintes passos(se certifique de que se o estado da sua instância é "running" ao longo desses passos):

Selecione Instance em target type
Dê um nome ao target group, insira 80 em Port e selecione a VPC criada anteriormente
image

Clique em "Next"
Selecione a instância criada anteriormente para ser inserida no Load Balancer, se certifique de que o valor em "Port for the selected instances" é 80 e clique em "Include as pending below"
Clique em Create target group
Agora, selecione a ferramenta Load Balancer no serviço de EC2, clique em Create load balancer e siga os seguintes passos:

Crie um Application Load Balancer
image

Dê um nome a ele, selecione a VPC criada anteriormente e todas as suas AZs disponíveis, mas certifique-se de escolher as subnets públicas(são as que possuem um internet gateway)
Selecione o security group criado anteriormente para o LB
Em Listeners and routing, selecione o protocolo HTTP e o target group que acabamos de criar
Clique em Create load balancer
Mais a frente nesse projeto, quando o Wordpress já estiver rodando na porta 80, poderemos acessá-lo no navegador por meio do DNS do load balancer, que você pode encontrar na aba Details do load balancer quando você clica nele.

RDS
O RDS é o serviço de bancos de dados relacionais da AWS. Com ele, nós vamos criar um banco de dados MYSQL para a nossa aplicação Wordpress. É possível usar o RDS no modo free tier, production ou dev/test, então o ideal é que você leia do que cada um se trata antes de seguir, mas já adianto que nesse projeto eu utilizei o free tier. Dito isso, procure e selecione o serviço de RDS no console AWS. No RDS, selecione Databases, clique em Create Database e siga os passos a seguir:

Selecione MYSQL em Engine type
Selecione a versão mais antiga disponível do MYSQL(isso geralmente garante que ele vai funcionar bem com o Wordpress)
Dê um nome à instância do seu DB(atenção: isso não é o nome do seu DB, mas sim da instância do DB na AWS) e crie um username e uma senha para o master user do DB
image

Selecione a VPC criada anteriormente, juntamente com o security group do DB e selecione uma AZ para o DB ser criado(se for sua primeira vez criando um DB, o DB subnet group provavelmente estará vazio, mas ele será criado automaticamente)
image

Nas configurações adicionais de conectividade, se certifique que a porta do banco de dados é 3306
image

Em Configurações adicionais, dê um nome ao seu DB
image

Clique em Create database
Script e EFS
Como iremos trabalhar com um auto scaling nesse projeto, seria inviável acessar individualmente cada uma das instâncias e baixar o docker, o docker-compose e montar o NFS. Por isso, utilizaremos um script que fará tudo isso por nós, excluindo a necessidade de fazermos qualquer acesso à instância desse tipo.

O script que iremos utilizar está disponível nesse repositório com o nome user_data.sh e seu conteúdo é esse aqui:
