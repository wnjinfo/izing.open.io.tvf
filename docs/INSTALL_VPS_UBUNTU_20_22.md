# Manual de Instação do izing.io na VPS

### Observação:
- Antes de começar a instalação é necessário ter criado antecipadamente os subdomínios e já estarem apontados para o IP da VPS.
- Feito ubuntu 20
- Nesse modelo vamos usar docker porque versão Postgresql do repositorio UBUNTU 20 ta dando erro com izing
- Senha usada 123@mudar
- Dominio Frontend: izing.seusite.com.br
- Dominio backend: backend.seusite.com.br
  
================================================

1. Alterando para root

```bash
sudo su root
```

2. Setar Time Zone para São Paulo e atualizar sistema

```bash
timedatectl set-timezone America/Sao_Paulo && apt update && apt upgrade -y
```

3. Reiniciar para atualizar kernel

```bash
reboot
```

4. Apos reniciar conectar no servidor novamente - Alterando para root

```bash
sudo su root
```

5. Intalar pacotes necessários

```bash
apt install -y ffmpeg fonts-ipafont-gothic fonts-wqy-zenhei fonts-thai-tlwg fonts-kacst fonts-freefont-ttf libxss1 apt-transport-https ca-certificates software-properties-common curl libgbm-dev wget unzip fontconfig locales gconf-service libasound2 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 libgcc1 libgconf-2-4 libgdk-pixbuf2.0-0 libglib2.0-0 libgtk-3-0 libnspr4 libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 ca-certificates fonts-liberation libappindicator1 libnss3 lsb-release xdg-utils python2-minimal build-essential libxshmfence-dev nginx
```

6. Adicionar repositorio Docker

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

7. Instalar docker

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

8. limpar pacotes não são mais usados

```bash
apt autoremove -y
```

9. Instalar POSTGRESQL no Docker

```bash
docker run --name postgresql -e POSTGRES_USER=izing -e POSTGRES_PASSWORD=123@mudar -e TZ="America/Sao_Paulo" -p 5432:5432 --restart=always -v /data:/var/lib/postgresql/data -d postgres
```

10. Instalar Redis no Docker

```bash
docker run --name redis-izing -e TZ="America/Sao_Paulo" -p 6379:6379 --restart=always -d redis:latest redis-server --appendonly yes --requirepass "123@mudar"
```


11. Instalar Portainer no Docker

```bash
docker run -d --name portainer -p 9000:9000 -p 9443:9443 --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```

12. baixar chave repositorio google crome

```bash
wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | sudo gpg --dearmour -o /usr/share/keyrings/chrome-keyring.gpg 
```

13. adicionar repositorio

```bash
sudo sh -c 'echo "deb [arch=amd64,arm64,ppc64el signed-by=/usr/share/keyrings/chrome-keyring.gpg] http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google.list'
```

14. update apt

```bash
sudo apt update 
```

15. instalar google crome

```bash
sudo apt install -y google-chrome-stable 
```

16. remover arquivo padrao nginx

```bash
rm /etc/nginx/sites-enabled/default
```


17. Criar o usário deploy

```bash
adduser deploy
```

18. Permisão sudo deploy
```bash
usermod -aG sudo deploy
```

19. Permisão docker deploy
```bash
usermod -aG docker deploy
```

20. Alterar para o novo usuário

```bash
su deploy
```

21. Realizar o download do node 20.x

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
```

22. Instalar o node

```bash
sudo apt-get install -y nodejs
```

23. Acessar o diretório raiz

```bash
cd ~
```

24. baixar o repositório do izing.open.io

```bash
git clone https://github.com/cleitonme/izing.open.io.git izing.io
```

25. Copiar o env de exemplo para o backend
```bash 
cp izing.io/backend/.env.example izing.io/backend/.env
```

26. Rodar o comando abaixo 2x para gerar JWT_SECRET e JWT_REFRESH_SECRET

```bash
openssl rand -base64 32
```

27. Editar os dados abaixo e colar os valores gerados no item 31.

```bash
# ambiente
NODE_ENV=dev

# URL do backend para construção dos hooks
BACKEND_URL=https://backend.seusite.com.br

# URL do front para liberação do cors
FRONTEND_URL=https://izing.seusite.com.br

# Porta utilizada para proxy com o serviço do backend
PROXY_PORT=443

# Porta que o serviço do backend deverá ouvir
PORT=3000


# conexão com o banco de dados
DB_DIALECT=postgres
DB_TIMEZONE=-03:00
DB_PORT=5432
POSTGRES_HOST=localhost
POSTGRES_USER=izing
POSTGRES_PASSWORD=123@mudar
POSTGRES_DB=postgres


# Chaves para criptografia do token jwt
JWT_SECRET=gerado no passo 31
JWT_REFRESH_SECRET=gerado no passo 31

# Dados de conexão com o REDIS
IO_REDIS_SERVER=localhost
IO_REDIS_PORT='6379'
IO_REDIS_DB_SESSION='2'
IO_REDIS_PASSWORD=123@mudar

#CHROME_BIN=/usr/bin/google-chrome
CHROME_BIN=/usr/bin/google-chrome-stable
#CHROME_BIN=null

# tempo para randomização da mensagem de horário de funcionamento
MIN_SLEEP_BUSINESS_HOURS=10000
MAX_SLEEP_BUSINESS_HOURS=20000

# tempo para randomização das mensagens do bot
MIN_SLEEP_AUTO_REPLY=4000
MAX_SLEEP_AUTO_REPLY=6000

# tempo para randomização das mensagens gerais
MIN_SLEEP_INTERVAL=2000
MAX_SLEEP_INTERVAL=5000


# dados do RabbitMQ / Para não utilizar, basta comentar a var AMQP_URL
RABBITMQ_DEFAULT_USER=admin
RABBITMQ_DEFAULT_PASS=123456
# AMQP_URL='amqp://admin:123456@host.docker.internal:5672?connection_attempts=5&retry_delay=5'

# api oficial (integração em desenvolvimento)
API_URL_360=https://waba-sandbox.360dialog.io

# usado para mosrar opções não disponíveis normalmente.
ADMIN_DOMAIN=izing.io

# Dados para utilização do canal do facebook
VUE_FACEBOOK_APP_ID=3237415623048660
FACEBOOK_APP_SECRET_KEY=3266214132b8c98ac59f3e957a5efeaaa13500

# Forçar utilizar versão definida via cache (https://wppconnect.io/pt-BR/whatsapp-versions/)
#WEB_VERSION=2.2409.2

# Customizar opções do pool de conexões DB
#POSTGRES_POOL_MAX=100
#POSTGRES_POOL_MIN=10
#POSTGRES_POOL_ACQUIRE=30000
#POSTGRES_POOL_IDLE=10000

# Limitar Uso do Izing Usuario e Conexões
USER_LIMIT=99
CONNECTIONS_LIMIT=99
```

28. Abrir para edição o arquivo .env com o comando abaixo e prencher com os dados acima. Para salvar se usa Ctrl + x

```bash
nano izing.io/backend/.env
```

29. Acessando pasta do backend

```bash
cd izing.io/backend
```

30. Instalando as dependências

```bash
npm install --force
```

31. Buildando o backend

```bash
npm run build
```

32. Reniciando docker
```bash
docker container restart postgresql
```

33. Reniciando docker
```bash
docker container restart redis-izing
```

34. Reniciando docker
```bash
docker container restart portainer
```

34. Criando as tabelas no BD

```bash
npx sequelize db:migrate
```

35. Inserindo dados em algumas tabelas do BD

```bash
npx sequelize db:seed:all
```

36. Instalando o PM2

```bash
sudo npm install -g pm2
```

37. Iniciando o backend com PM2

```bash
pm2 start dist/server.js --name izing-backend
```

38. Gerar Startup

```bash
pm2 startup ubuntu -u deploy
```

39. Gerar status parte 2

```bash
sudo env PATH=$PATH:/usr/bin pm2 startup ubuntu -u deploy --hp /home/deploy
```

40. Acessando pasta do frontend

```bash
cd ../frontend
```

41. copiando .env do example

```bash
cp .env.example .env
```

42. Editando o arquivo .env com o comando abaixo e prencher com os dados do item 42. Para salvar se usa Ctrl + x

```bash
nano .env
```

43. Dados env frontend

```bash
VUE_URL_API='https://backend.seusite.com.br'
VUE_FACEBOOK_APP_ID='23156312477653241'
```

44. Criar arquivo server.js com dados do item 44. Para salvar se usa Ctrl + x

```bash
nano server.js
```

45. Dados para gerar server.js
```bash
// simple express server to run frontend production build;
const express = require('express')
const path = require('path')
const app = express()
app.use(express.static(path.join(__dirname, 'dist/pwa')))
app.get('/*', function (req, res) {
  res.sendFile(path.join(__dirname, 'dist/pwa', 'index.html'))
})
app.listen(4444)
```

46. Instalando as dependências
```bash
npm install --force
```

47. Instalando Quasar

```bash
npm i @quasar/cli
```

48. Buildando o frontend

```bash
npm run build
```

49. Iniciando o frontend com PM2
```bash
pm2 start server.js --name izing-frontend
```

50. Salvando os serviços iniciados pelo PM2

```bash
pm2 save
```

51. Listar os serviços iniciados pelo PM2

```bash
pm2 list
```

52. Editar os dados abaixo com a URL que será usada para acessar o frontend.

```bash
server {
  server_name izing.seusite.com.br;

  location / {
    proxy_pass http://127.0.0.1:4444;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_cache_bypass $http_upgrade;
  }

}
```

53. Criar e editar o arquivo izing-frontend com o comando abaixo e prencher com os dados do item 52. Para salvar se usa Ctrl + x

```bash
sudo nano /etc/nginx/sites-available/izing-frontend
```

54. Editar os dados abaixo com a URL que será usada para acessar o backend.

```bash
server {
  server_name backend.seusite.com.br;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_cache_bypass $http_upgrade;
  }

}
```

55. Criar e editar o arquivo izing-frontend com o comando abaixo e prencher com os dados do item 54. Para salvar se usa Ctrl + x

```bash
sudo nano /etc/nginx/sites-available/izing-backend
```

56. Criar link simbólico para o arquivo izing-frontend

```bash
sudo ln -s /etc/nginx/sites-available/izing-frontend /etc/nginx/sites-enabled/
```


57. Criar link simbólico para o arquivo izing-backend

```bash
sudo ln -s /etc/nginx/sites-available/izing-backend /etc/nginx/sites-enabled/
```

58. Editar o arquivo de configuração do nginx com o comando abaixo e prencher com os dados do item 59. adicionar antes# server_tokens off;. Para salvar se usa Ctrl + x

```bash
sudo nano /etc/nginx/nginx.conf
```

59. adicionar antes# server_tokens off;

```bash
underscores_in_headers on;	
client_max_body_size 100M;
large_client_header_buffers 16 5120k;
```
60. Testar as configurações do nginx

```bash
sudo nginx -t
```

61. Restartar o nginx

```bash
sudo service nginx restart
```

62. Instalar o suporte a pacotes Snap

```bash
sudo apt-get install snapd
```

63. Instalar o pacote do notes

```bash
sudo snap install notes
```

64. Instalar o pacote do certbot(SSL)

```bash
sudo snap install --classic certbot
```

65. Gerar certificado

```bash
sudo certbot --nginx
```

65. reniciar serviços docker
```bash
docker container restart portainer
```

66. reniciar serviços docker
```bash
docker container restart postgresql
```

67. reniciar serviços docker
```bash
docker container restart redis-izing
```

Pronto sistema instalado so acessar frontend

Usuário painel SaaS

Usuário

super@izing.io  

Senha:

123456

Usuário padrão para acesso

Usuário

admin@izing.io  

Senha:

123456


Problemas conexão?

1. Outra versão js pode se tentar
Na pasta backend execute
```bash
npm r whatsapp-web.js
```
```bash
npm i whatsapp-web.js@^1.25.0
```
```bash
rm .wwebjs_auth -Rf
```
```bash
rm .wwebjs_cache -Rf
```
```bash
pm2 restart all
```

2. Versão EXODUS
Na pasta backend execute
```bash
npm r whatsapp-web.js
```
```bash
npm install github:pedroslopez/whatsapp-web.js#webpack-exodus
```
```bash
rm .wwebjs_auth -Rf
```
```bash
rm .wwebjs_cache -Rf
```
```bash
pm2 restart all
```

Para reinstalar o whatsapp.js.. verifique no repositorio oficial se não tem alguma mais atual

## Erros

"Internal server error: SequelizeConnectionError: could not open file \"global/pg_filenode.map\": Permission denied"

```bash
docker container restart postgresql
```
```bash
docker exec -u root postgresql bash -c "chown -R postgres:postgres /var/lib/postgresql/data"
```
```bash
docker container restart postgresql
```

## Acesso Portainer gerar senha
"Your Portainer instance timed out for security purposes. To re-enable your Portainer instance, you will need to restart Portainer."

```bash
docker container restart portainer
```

Depois acesse novamente url http://seuip:9000/


## Recomendação de VPS boa e barata

-  [Powerful cloud VPS & Web hosting.](https://control.peramix.com/?affid=58)

- Cupom 25% desconto "WHAZING"

```bash
WHAZING
```

## Consultoria particular

Para quem gostaria de uma consultoria ou que eu faça instalação pode chamar no whatsapp (será cobrado por isso) 48 999416725

-  [Versão baseada no izing com Painel SaaS e API Baileys](https://github.com/cleitonme/Whazing-SaaS)
