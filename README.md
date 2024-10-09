# Guia Completo para Implantar o Keycloak na Nuvem 
O keycloak será utilizado para o Gerenciamento de Acesso ao Aplicativo Monitora TV.

### keycloak-aws
Keycloak é uma solução de gerenciamento de identidade e acesso de código aberto. [Saiba mais](https://www.keycloak.org/).

## Referencia
Configurando AWS EC2: PostgreSQL, Keycloak y NGINX. [Ver video](https://www.youtube.com/watch?v=ErpFhcHsEXc&list=PLlYjHWCxjWmApyeC9KZa9m8otfcJi9-NQ&index=5/).


## Pré-requisitos
- Uma conta em um provedor de serviço na nuvem (ex: AWS).
- Um domínio registrado.
- Conhecimentos básicos em AWS e gerenciamento de servidores.

## 1 - Instalação da Infraestrutura

### AWS VPC - Criar VPC
Amazon Virtual Private Cloud [Saiba mais](https://aws.amazon.com/pt/vpc/). 

### Criar Subnet
- Uma pública(Acesso ao keycloak e ao Monitora TV frontend) e outra privada(acesso privado ao backend).

### Criar Instância EC2
- Nome: `monitoratv-public`
- Protocólos e portas: configurar regras de entrada conforme necessário.

### Criar Grupo de Segurança
- Nome: `monitoratv-sg`
- Regras de entrada: configurar de acordo com os requisitos.

### Criar IP Elástico
- IP: `<IP elastico criado>`
- Alocar o IP elástico à instância criada (`monitoratv-public`).

### Route 53 - Criar Subdomínio
1. Acessar Route 53 > Hosted Zone > `<dominio>.com`
2. Criar registro `keycloak.<dominio>.com` do tipo "A" e definir o valor da rota como o IP elástico `<IP elastico criado>`.

## 2 - Criar Banco de Dados
1. Acessar AWS RDS > Criar banco de dados > Postgres.
   - Identificador da instância DB: `<nome da instancia>`
   - Nome de usuário: `postgres`
   - Grupo de segurança: `postgres-sg`
   - Selecionar t3 gratuito.
   - Armazenamento: 20 GB.
2. Criar banco de dados "keyclaok", a estrutura inicial do banco será criada na primera compilação do Keycloak.

## 3 - Configurar Keycloak

### Atualizar Instância `monitoratv-public`
1. Acessar a instância:
   ```bash
   ssh -i "<sua-chave.pem>" ubuntu@<IP elastico criado>
   chmod 400 "<sua-chave.pem>"
   ssh -i "<sua-chave.pem>" ubuntu@<IP elastico criado>
   ```
2. Atualizar pacotes:
   ```bash
   sudo apt update
   sudo apt upgrade -y
   ```

### Instalar Java JDK 21
```bash
sudo apt install openjdk-21-jdk
```

### Instalar Keycloak
```bash
wget https://github.com/keycloak/keycloak/releases/download/24.0.2/keycloak-24.0.2.tar.gz
tar -xzf keycloak-24.0.2.tar.gz
sudo rm keycloak-24.0.2.tar.gz
sudo mv keycloak-24.0.2 /opt/keycloak
sudo groupadd keycloak
sudo useradd -r -g keycloak -d /opt/keycloak/ -s /sbin/nologin keycloak
cd /opt
sudo chown -R keycloak: keycloak
sudo chmod -R o+x /opt/keycloak/bin/
```

### Configurar Keycloak
1. Editar o arquivo de configuração do Keycloak:
   ```bash
   sudo nano /opt/keycloak/conf/keycloak.conf
   ```
   - Configurações conectar e criar estrutura do banco:
     ```properties
     db=postgres
     db-username=postgres
     db-password=<password>
     db-url=jdbc:postgresql://<Endpoint-aws>:5432/keycloak
     hostname=keycloak.<dominio>.com
     http-port=8080
     http-relative-path=/auth
     ```

### Adicionar Grupo de Segurança para a Instância
1. Acessar EC2 > Segurança > Grupos de segurança:
   - Regras de entrada:
     - TCP 8080 (0.0.0.0/0)
     - TCP 443 (0.0.0.0/0)
     - TCP 22 (0.0.0.0/0)
     - TCP 80 (0.0.0.0/0)

### Executar Keycloak pela Primeira Vez
A primeira compilação "build" vai gerar a estrutura inicial do banco de dados "Keycloak".

```bash
sudo /opt/keycloak/bin/kc.sh build
```
- Configurar variáveis de ambiente:
```bash
export KEYCLOAK_ADMIN=admin
export KEYCLOAK_ADMIN_PASSWORD=admin@xxxxxx
```
- Executar em modo de desenvolvimento:
```bash
sudo -E /opt/keycloak/bin/kc.sh start-dev
```

### Testar Acesso
- URL: `http://keycloak.<dominio>.com:8080/auth`
- Usuário: `admin`
- Senha: `admin@102030`

### Desabilitar SSL
```sql
update keycloak.public.realm set ssl_required = 'NONE' where "name"='master';
```

### Criar Serviço Keycloak
1. Criar arquivo de serviço:
   ```bash
   sudo nano /etc/systemd/system/keycloak.service
   ```
   - Conteúdo:
     ```ini
     [Unit]
     Description=The Keycloak server
     After=syslog.target network.target
     Before=httpd.service

     [Service]
     User=keycloak
     Group=keycloak
     SuccessExitStatus=0 143
     ExecStart=/opt/keycloak/bin/kc.sh start-dev
     ExecStop=/opt/keycloak/bin/kc.sh stop
     Restart=always
     RestartSec=3

     [Install]
     WantedBy=multi-user.target
     ```

2. Testar:
```bash
sudo systemctl daemon-reload
sudo systemctl enable keycloak
sudo systemctl start keycloak
sudo systemctl status keycloak
```

### Criar Certificado Keycloak
1. Instalar Certbot:
```bash
sudo apt install certbot -y
```
2. Solicitar Certificado:
```bash
sudo certbot certonly --standalone --preferred-challenges http -d keycloak.<dominio>.com
```

### Referenciar Certificado no Keycloak
1. Editar arquivo de configuração do Keycloak:
Exemplo: conf/keycloak.conf
```bash
sudo nano /opt/keycloak/conf/keycloak.conf
```
   - Adicionar:
     ```properties
     https-certificate-file=/etc/letsencrypt/live/keycloak.<dominio>.com/fullchain.pem
     https-certificate-key-file=/etc/letsencrypt/live/keycloak.<dominio>.com/privkey.pem
     hostname=keycloak.<dominio>.com
     https-port=8443
     http-relative-path=/auth
     ```

2. Reiniciar o Keycloak:
```bash
sudo /opt/keycloak/bin/kc.sh build
```

### Alterar Serviço para Modo Produção
1. Editar serviço:
```bash
sudo nano /etc/systemd/system/keycloak.service
```
   - Mudar `ExecStart` para:
     ```bash
     ExecStart=/opt/keycloak/bin/kc.sh start
     ```

2. Aplicar alterações:
```bash
sudo systemctl daemon-reload
sudo systemctl restart keycloak
```

### Testar Acesso
- URL: `https://keycloak.<dominio>.com/auth/admin/master/console/`
- Usuário: `admin`
- Senha: `xxxxxx`

## 4 - Instalar Nginx
1. Instalar Nginx na EC2 pública:
```bash
sudo apt update && sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl status nginx
```

### Configurar Nginx
1. Editar configuração:
Exemplo: conf/nginx.conf

```bash
sudo nano /etc/nginx/nginx.conf
```
   - Adicionar:
     ```nginx
     server {
         listen 443 ssl http2;
         server_name keycloak.<dominio>.com;

         ssl_certificate "/etc/letsencrypt/live/keycloak.<dominio>.com/fullchain.pem";
         ssl_certificate_key "/etc/letsencrypt/live/keycloak.<dominio>.com/privkey.pem";

         location /auth/ {
             proxy_pass https://keycloak.<dominio>.com:8443/auth/;
         }

         location / {
             proxy_pass https://keycloak.<dominio>.com:8443/auth/;
         }
     }
     ```

2. Aplicar alterações:
```bash
sudo systemctl daemon-reload
sudo nginx -s reload
```

### Testar Nginx
- URL: `https://keycloak.<dominio>.com/auth`

## 5 - Configurar Acesso no Keycloak

1. Acessar Keycloak:
- URL: `https://keycloak.<dominio>.com/auth`
- Usuário: `admin`
- Senha: `xxxxx`


Homologação: https://monitoratv.gustavokanashiro.com
- URL: `https://keycloak.<dominio>.com/auth`
- Usuário: `admin`
- Senha: `admin@102030`


### Criar Realm
- Nome: `monitoratv`

### Criar Roles
- Role: `ADMIN` > `Admin user`

### Criar Usuário
- Nome: `usr_admin`
- Roles: `BASIC` e `ADMIN`
- Senha: `102030`

### Criar Client
1. Client: `localhost-dev`
   - Client ID: `localhost-dev`
   - Nome: `Client localhost para ambiente de desenvolvimento`
   - URL raiz: `http://localhost`
   - URL inicial: `http://localhost`
   - URIs de redirecionamento válidas: `http://localhost/*`
   - Autenticação do cliente: Ativado.
   - Fluxos de autenticação: 
     - Standard flow
     - Direct access grants


2. Client: `frontend-monitoratv`
   - Client ID: `frontend-monitoratv`
   - Nome: `Client para ambiente de produção`
   - URIs de redirecionamento válidas: `https://monitoratv.gustavokanashiro.com/*`
   - Web Origrns: `https://monitoratv.gustavokanashiro.com/*`
   - Autenticação do cliente: Ativado.
   - Fluxos de autenticação: 
     - Standard flow
     - Direct access grants
 
 
### Configurar Keycloak 
O video abaixo explica como criar realm, client, usuários entre outras funcionalidades do keycloak.
https://www.youtube.com/playlist?list=PL-XEb3GK7JUrzMDzRAiRG-AV8iP73FF8