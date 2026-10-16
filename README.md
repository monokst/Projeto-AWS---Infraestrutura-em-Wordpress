# Documentação – Infraestrutura WordPress na AWS

## Sobre o projeto
Este projeto tem como objetivo hospedar uma aplicação WordPress na AWS, utilizando serviços de computação, banco de dados e rede. A infraestrutura foi desenvolvida como um projeto prático de estudo em Cloud Computing.

## Serviços utilizados
- Amazon VPC: criação de uma rede virtual isolada.
- Sub-redes públicas e privadas: organização dos recursos de acordo com sua exposição à internet. A EC2 fica em uma subnet pública e o RDS nas subnets privadas.
- Internet Gateway: conectividade com a internet para os recursos da subnet pública.
- Tabela de rotas: direciona o tráfego da subnet pública (0.0.0.0/0) para o Internet Gateway.
- Amazon EC2: hospedagem do WordPress com Apache e PHP.
- Elastic IP: endereço IPv4 público fixo associado à EC2, usado para acessar o WordPress.
- Amazon EBS: volume gp3 de 15 GB como disco da EC2.
- IAM e Systems Manager (SSM): função com a política AmazonSSMManagedInstanceCore, para acesso à EC2 pelo console, sem abrir a porta 22.
- Amazon RDS: gerenciamento do banco de dados MySQL, em subnets privadas e sem acesso público.
- Security Groups: controle do tráfego. A EC2 recebe HTTP (TCP 80) apenas do meu IP; o RDS aceita MySQL (TCP 3306) apenas do Security Group da EC2.

## Arquitetura
A arquitetura foi desenvolvida na AWS com o objetivo de hospedar uma aplicação WordPress, utilizando uma instância Amazon EC2 para executar a aplicação e o Amazon RDS com MySQL para armazenar os dados.
A infraestrutura utiliza uma Amazon VPC para organizar os recursos de rede, com sub-redes públicas e privadas para separar os componentes. Os Security Groups controlam o tráfego de entrada e saída, contribuindo para a segurança do ambiente. A infraestrutura possui quatro subnets, distribuídas em duas zonas de disponibilidade da região us-east-1 (N. Virginia): duas públicas e duas privadas. Uma subnet pública e uma privada estão na zona us-east-1a, enquanto as outras duas estão na zona us-east-1b.
Na instância EC2, foram configurados o Apache, o PHP e o WordPress em um ambiente Linux, além da integração com o banco de dados MySQL hospedado no Amazon RDS. A EC2 está em uma subnet pública e utiliza um Elastic IP. Seu Security Group permite conexões HTTP pela porta 80 somente a partir do meu IP público (/32). Não há regra de entrada para SSH (porta 22), pois o acesso administrativo é realizado pelo AWS Systems Manager (SSM).
O RDS está configurado sem acesso público e utiliza um grupo de sub-redes composto por duas subnets privadas. O acesso ao banco de dados pela porta MySQL 3306 é permitido somente a partir do Security Group da EC2. Não foi utilizado NAT Gateway, com o objetivo de reduzir custos, já que o RDS não precisa de acesso direto à internet para receber conexões da aplicação hospedada na EC2.
O projeto foi desenvolvido de forma prática, permitindo explorar conceitos de computação em nuvem, redes, administração Linux, hospedagem de aplicações web, conectividade com banco de dados e segurança na AWS.


## Custos estimados
Os custos dependem da região AWS, do tempo de utilização e das configurações de cada recurso.
- Amazon VPC: sem cobrança pela criação da VPC.
- Subnets: sem cobrança pela criação das subnets.
- Security Groups: sem cobrança pela criação dos grupos de segurança.
- Internet Gateway: sem cobrança pelo gateway em si; o tráfego e outros recursos associados podem gerar custos.
- Amazon EC2: instância t3.micro. A cobrança depende das condições de elegibilidade e do plano da conta.
- Amazon EBS: volume gp3 de 15 GB, cobrado por GB provisionado, conforme a elegibilidade da conta.
- Elastic IP: a AWS cobra por hora por cada endereço IPv4 público, esteja ele associado ou não, conforme a elegibilidade da conta. Por isso, o Elastic IP deve ser liberado ao encerrar o projeto.
- Amazon RDS: instância db.t4g.micro, com armazenamento de 20 GiB. Pode haver cobrança pela instância, pelo armazenamento, pelos backups e por outros recursos utilizados.
- NAT Gateway: não utilizado, para evitar custo adicional.


## Etapas de implementação

### 1.	Criação da VPC e Subnet
O primeiro passo foi a criação da VPC. Nela, coloquei o CIDR IPv4 como 10.0.0.0/16, um bloco muito utilizado em testes, com mais de 65 mil (65.536) IPs disponíveis para os recursos.
Em seguida, criei as subnets, que são como "caixinhas" dentro da rede e podem ser privadas ou públicas. Como padrão, criamos a mesma quantidade de subnets públicas e privadas; no meu caso, duas públicas e duas privadas. Todas usam o range /24, que disponibiliza 251 IPs utilizáveis por subnet (a AWS reserva 5 endereços em cada uma). A EC2 foi posicionada em uma subnet pública e o RDS nas subnets privadas.

### 2.	Criação do Internet Gateway
Para a subnet pública ter acesso à internet, é necessário criar um Internet Gateway e anexá-lo à VPC. Em seguida, criei a Tabela de rotas com a rota 0.0.0.0/0 apontando para o Internet Gateway e a associei às subnets públicas. As subnets privadas não possuem essa rota, e não foi utilizado NAT Gateway, para evitar custos, já que o RDS não precisa de acesso à internet.

### 3.	EC2
Para criar a EC2, utilizei a imagem Amazon Linux 2023, elegível ao nível gratuito, e a instância t3.micro, posicionada em uma subnet pública. Criei um par de chaves apenas para a instância. O Security Group possui uma única regra de entrada: HTTP, TCP 80, com origem no meu IP (formato x.x.x.x/32), por segurança. Não há regra de entrada para SSH (porta 22), pois o acesso é feito pelo SSM. A saída ficou como padrão (todo o tráfego liberado), o que permite instalar pacotes e conectar ao RDS. O disco EBS foi configurado com 15 GB, tipo gp3.
Obs: Para podermos nos conectar à instância pelo console, via SSM, recomendo criar uma função no IAM com a política AmazonSSMManagedInstanceCore.

**O passo a passo é:**
- IAM  Funções
- Tipo de entidade confiável  Serviço da AWS
- Caso de uso  EC2
- Caso de uso especificado  EC2 Role for AWS Systems Manager
- Próximo  Nomear
- Criar perfil


Voltando na criação da instância, em Detalhes avançados → Perfil de instância do IAM, selecione a função criada. Depois, clique em Executar instância.
Com a instância em execução, aloque um Elastic IP (EC2 → Elastic IPs → Alocar endereço IP elástico) e associe-o à instância. Assim, o endereço de acesso ao WordPress não muda ao parar e iniciar a EC2, e terá acesso ao IPV4 Público da máquina.

### 4.	Criar o RDS
O Amazon RDS (Relational Database Service) é um serviço da AWS que facilita a criação e o gerenciamento de bancos de dados. Em vez de instalar e administrar o banco manualmente na EC2, podemos utilizar o RDS para manter os dados em um serviço separado.
Neste projeto, utilizei o MySQL como mecanismo do banco de dados. O WordPress se conecta a ele para armazenar informações como páginas, publicações, usuários e configurações do site.




#### 1. Acessar o Amazon RDS
No console da AWS, pesquise por Amazon RDS e acesse o serviço. No menu lateral, selecione Bancos de dados e clique em Criar banco de dados.

#### 2. Escolher o método de criação
Na opção de método de criação, selecione Configuração completa. Esse método permite configurar os detalhes do banco de dados antes de criá-lo.

#### 3. Escolher o mecanismo e o modelo
Na seção de opções de mecanismo, selecione MySQL.

Em seguida, escolha o modelo Nível gratuito, caso esteja disponível para sua conta e configuração. Esse modelo é destinado a ambientes de testes e aprendizado, mas a elegibilidade e os custos dependem das condições atuais da AWS e dos recursos selecionados.

#### 4. Configurar as informações do banco
Na seção de configurações, preencha os campos:

- Identificador da instância de banco de dados: defina um nome para identificar o banco no console da AWS.
- Nome de usuário principal: defina o usuário administrador do banco de dados, por exemplo, admin.
- Gerenciamento de credenciais: selecione Autogerenciada.
- Senha principal: crie uma senha forte para o usuário principal e guarde-a em um local seguro.

O identificador é o nome utilizado para localizar a instância no console. Já o usuário e a senha são as credenciais utilizadas para autenticar as conexões com o banco.

#### 5. Configurar a instância e o armazenamento
Na seção de configuração da instância, já vem como padrão a instância do tipo db.t4g.micro, por estarmos Nível gratuito.

Em armazenamento, selecione o tipo SSD de uso geral (gp3) e pode deixar como o tamanho padrão como 20 GiB.

O tipo de instância determina os recursos de processamento e memória disponíveis para o banco. Já o armazenamento define o espaço utilizado para guardar os dados.

Antes de criar o banco, confira se o tipo de instância e o armazenamento selecionados são compatíveis com o mecanismo e a região escolhidos.

#### 6. Configurar a conectividade
Na seção Conectividade, configure os seguintes campos:

- Nuvem privada virtual (VPC): selecione a mesma VPC utilizada pela instância EC2.
- Grupo de sub-redes (DB subnet group): selecione um grupo de sub-redes que contenha apenas as duas subnets privadas.
- Acesso público: selecione Não, para que o banco não receba IP público e fique acessível apenas dentro da VPC.

Recomendamos criar um novo grupo de sub-redes pelas opções laterais do console do RDS, em vez de usar o padrão. Durante a criação, selecione a VPC e apenas as duas subnets privadas, em zonas de disponibilidade diferentes.

O grupo de sub-redes informa ao RDS em quais subnets ele poderá posicionar a instância. Para um ambiente mais seguro, prefira subnets privadas. O RDS normalmente exige subnets em pelo menos duas zonas de disponibilidade para o grupo de sub-redes.

#### 7. Configurar o Security Group do banco de dados
Na configuração de conectividade, escolha a opção para criar um novo grupo de segurança ou selecione um grupo existente destinado ao RDS.

O Security Group funciona como um firewall virtual que controla quais conexões podem chegar ao banco de dados.

Configure uma regra de entrada com os seguintes dados:
- Tipo: MySQL/Aurora.
- Protocolo: TCP.
- Porta: 3306.
- Origem: Security Group da instância EC2 (e não o meu endereço IP).
Ao utilizar o Security Group da EC2 como origem, você permite que as conexões partam da instância autorizada, sem precisar liberar a porta do banco para todos os endereços IP da internet.

A EC2 e o RDS também precisam ter conectividade de rede entre si para que essa comunicação funcione.

#### 8. Configurações adicionais
Na seção Configurações adicionais, localize a opção relacionada à habilitação de backups automatizados.
Para este projeto de estudo, você pode desabilitar essa opção para evitar a retenção de backups automatizados, desde que não precise deles.

Atenção: desabilitar os backups automatizados reduz as possibilidades de recuperação dos dados. Em um ambiente de produção, é recomendável manter uma estratégia de backup adequada.

#### 9. Criar o banco de dados
Após revisar todas as configurações, clique em Criar banco de dados.

### 5.	Instalação do Apache, PHP e WordPress e integração com o Amazon RDS

#### Passo 1 — Atualizar o sistema
```bash
sudo dnf update -y
```

Atualiza os pacotes instalados no sistema operacional.

#### Passo 2 — Instalar e iniciar o Apache
```bash
sudo dnf install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```

- install: instala o Apache.
- start: inicia o serviço.
- enable: configura a inicialização automática junto com a EC2.
- status: verifica se o serviço está funcionando.


#### Passo 3 — Instalar o PHP e as extensões necessárias
```bash
sudo dnf install -y php php-mysqlnd php-fpm php-json php-gd php-mbstring php-xml php-curl
php -v
```

O PHP executa o código do WordPress. As extensões permitem, entre outras coisas, a conexão com o banco de dados e o processamento de imagens.
O comando php -v verifica a versão instalada.

#### Passo 4 — Baixar e instalar o WordPress
```bash
cd /tmp
wget <https://wordpress.org/latest.tar.gz>
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/
```

- cd /tmp: acessa o diretório temporário.
- wget: baixa o WordPress.
- tar -xzf: extrai os arquivos.
- cp -r: copia os arquivos para o diretório utilizado pelo Apache.

#### Passo 5 — Ajustar as permissões

```bash
sudo chown -R apache:apache /var/www/html/
sudo chmod -R 755 /var/www/html/
```

O primeiro comando define o proprietário e o grupo dos arquivos como apache. O segundo ajusta as permissões de acesso.
Para produção, é recomendável revisar essas permissões para evitar conceder acesso de escrita desnecessário ao servidor web (por exemplo, diretórios com 755 e arquivos com 644).


#### Passo 6 — Configurar o WordPress para utilizar o RDS

Crie o arquivo de configuração:
```bash
cd /var/www/html/
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

Localize as configurações do banco de dados e preencha com os dados do seu RDS:
```bash
define( 'DB_NAME', 'NOME-DO-BANCO' );
define( 'DB_USER', 'USUARIO' );
define( 'DB_PASSWORD', 'SUA_SENHA_SEGURA' );
define( 'DB_HOST', 'SEU-ENDPOINT-RDS' );
```


Substitua os valores pelos dados reais:
- DB_NAME: nome do banco de dados criado no RDS.
- DB_USER: usuário criado no RDS.
- DB_PASSWORD: senha desse usuário.
- DB_HOST: endpoint do RDS, incluindo a porta se necessário, por exemplo meu-banco.xxxxxxxxxxxx.sa-east-1.rds.amazonaws.com:3306 para MySQL ou MariaDB.

Gere também as chaves de segurança do WordPress em:
Gerar chaves de segurança do WordPress

Substitua o bloco AUTH_KEY até NONCE_SALT pelas chaves geradas. Salve o arquivo no Nano com Ctrl + O, Enter e saia com Ctrl + X.


Importante: o banco de dados e o usuário precisam existir no RDS antes de finalizar a instalação. O usuário deve ter as permissões necessárias sobre o banco do WordPress.

#### Passo 7 — Reiniciar o Apache e verificar o serviço
```bash
sudo systemctl restart httpd
sudo systemctl status httpd
```

#### Passo 8 — Testar a conectividade com o RDS e criar o banco
Instale o cliente MySQL (MariaDB) na EC2
```bash
sudo dnf install mariadb105 -y
```

Verifique a versão
```bash
mysql --version
```

Entre dentro do banco:
```bash
mysql -h ENDPOINT-DO-RDS -P 3306 -u USUARIO -p
```

Logo após clicar em Enter, aparecerá para inserir a senha.


Assim que entrar no banco, insira os seguintes comandos:
```bash
SHOW DATABASES;
```

```bash
CREATE DATABASE wordpress;
```

```bash
SHOW DATABASES;
```

```bash
EXIT;
```

#### Passo 9 — Verificar o Apache
Para verificar se o Apache está respondendo na própria EC2, use:
```bash
curl -I http://localhost
```
Se aparecer status code 302, o WordPress está respondendo e redirecionando para a tela de instalação.


#### Passo 10 — Acessar a interface do WordPress

Para verificar se o a tela do Wordpress está funcionando
http://ELASTIC-IP-DA-EC2/wp-admin/install.php


## Observações e limitações
- Acesso restrito: o site só responde ao IP liberado no Security Group. Se o meu IP público mudar, é preciso atualizar a regra de entrada da EC2.
- O acesso é feito por HTTP, sem HTTPS, por se tratar de um ambiente de estudo sem domínio.
- As credenciais do banco ficam em texto no wp-config.php. Em produção, o recomendado é utilizar o AWS Secrets Manager.
- Os backups automatizados do RDS foram desabilitados para este estudo; em produção, mantenha uma estratégia de backup.

## Limpeza dos recursos
- Encerrar (terminate) a instância EC2.
- Liberar o Elastic IP, para interromper a cobrança do IPv4 público.
- Excluir a instância RDS (sem snapshot final, se os dados não forem necessários).
- Excluir os Security Groups, o DB subnet group, a tabela de rotas, as subnets e, por fim, desanexar e excluir o Internet Gateway e a VPC.
