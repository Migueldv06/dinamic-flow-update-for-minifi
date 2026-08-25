# Criando pastas

mkdir nifi
mkdir minifi

nano nifi/docker-compose.yaml
services:
  nifi:
    build: .
    container_name: nifi
    restart: unless-stopped

    ports:
      - "8443:8443"   # HTTPS

    environment:
      # Web / HTTPS
      NIFI_WEB_HTTPS_HOST: 0.0.0.0
      NIFI_WEB_HTTPS_PORT: 8443

      # Single user auth (login inicial)
      SINGLE_USER_CREDENTIALS_USERNAME: admin
      SINGLE_USER_CREDENTIALS_PASSWORD: sejalivrenifi2026

    volumes:
      # Configuração
      - ./conf:/opt/nifi/nifi-current/conf

      # Logs
      - ./logs:/opt/nifi/nifi-current/logs

      # Repositórios (estado do NiFi)
      - ./state:/opt/nifi/nifi-current/state
      - ./database_repository:/opt/nifi/nifi-current/database_repository
      - ./flowfile_repository:/opt/nifi/nifi-current/flowfile_repository
      - ./content_repository:/opt/nifi/nifi-current/content_repository
      - ./provenance_repository:/opt/nifi/nifi-current/provenance_repository

    networks:
      - nifi-net

networks:
  nifi-net:
    driver: bridge

nano nifi/Dockerfile
FROM apache/nifi:2.11.0
USER root
RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
USER nifi

# criando pastas para o nifi
cd nifi
mkdir conf logs state database_repository flowfile_repository content_repository provenance_repository
sudo chown -R 1000:1000 conf logs state database_repository flowfile_repository content_repository provenance_repository

# primeira execução do nifi
comente a linha 
- ./conf:/opt/nifi/nifi-current/conf
no arquivo docker-compose.yaml

docker compose up -d

sudo docker cp nifi:/opt/nifi/nifi-current/conf/. ./conf/
sudo chown -R 1000:1000 ./conf

docker compose down

descomente a linha
- ./conf:/opt/nifi/nifi-current/conf

docker compose up -d

acesse https://localhost:8443/nifi/ 

login: admin
senha: sejalivrenifi2026

# Criando github
Crie um repositorio no github, podendo ser privado por segurança
crie um arquivo README.md
crie uma token de acesso no github em Settings/Developer settings/Personal access tokens/Generate new token (classic)
gere um tokem com permissão de repositorios

# Conectando o nifi ao github
no nifi acesse no menu superior direito nas 3 barrinhas Controller Settings > Registry Clients e crie um "GitHubFlowRegistryClient" e edite as seguintes configurações:
Ex:
Repository Owner: Migueld06
Repository Name nifi-teste
Authentication Type: Personal Access Token
Personal Access Token: ********

agora no seu proccess group clique com o botão direito e vá em "Version" > "Start version control" e selecione o "GitHubFlowRegistryClient" criado anteriormente
Informe um flow name, ex "NiFi-Flow" e clique em "Save"

# Preparando github para converter o fluxo
git clone git@github.com:Migueldv06/nifi-teste.git

baixe o minifi toolkit

cd nifi-teste

wget https://dlcdn.apache.org/nifi/2.11.0/minifi-toolkit-2.11.0-bin.zip
unzip minifi-toolkit-2.11.0-bin.zip
rm minifi-toolkit-2.11.0-bin.zip

agora no github Actions na opção simple workflow clique em configure
crie o arquivo gerar-flow-minifi.yml com o codigo:
name: Gerar Flow MiNiFi

on:
  push:
    branches:
      - main
    paths:
      - '**.json'
      - '**.xml'
      - 'minifi-toolkit-2.11.0/**'

jobs:
  build-and-convert:
    runs-on: ubuntu-latest
    # Concede permissão de escrita para salvar o commit no repositório
    permissions:
      contents: write

    steps:
      - name: Checkout do código
        uses: actions/checkout@v4

      - name: Configurar Java 21
        uses: actions/setup-java@v5
        with:
          distribution: 'temurin'
          java-version: '21'

      - name: Dar permissão aos scripts do Toolkit
        run: chmod -R +x ./minifi-toolkit-2.11.0/bin/

      - name: Executar conversão
        run: ./minifi-toolkit-2.11.0/bin/config.sh transform-nifi ./default/NiFi-Flow.json ./default/flow-minifi.json

      # Salva e envia o arquivo atualizado para o repositório
      - name: Commitar e enviar alterações
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "chore: atualizar flow-minifi.json via pipeline [skip ci]"
          file_pattern: 'default/flow-minifi.json'

faça o pull do git e suba suas alterações para o github
git pull && git add . && git commit -m "adicionando minifi toolkit" && git push

lembrese de conferir o nome dos arquivos utilizados no workflow
agora va para o nifi, edite o workflow, salve, e veja se é gerado o arquivo:
default/flow-minifi.json
na pasta do seu repositório 

# Criando fluxos
crie um process group teste
adicione um GenerateFlowFile
e adicione parametros de teste apenas
lige ele a um LogAttribute

e crie outro fluxo com um ExecuteProcess
mude o scheduler para 1 minuto
e configure:
Command: git
Command Arguments: pull origin main -q
Working Directory: ./conf/nifi-teste

o ExecuteProccess vai dar alerta pois a pasta ./conf/nifi-teste não existe.
Eu irei clonar o repositório do github para dentro da pasta conf/nifi-teste no servidor para que o ExecuteProcess funcione corretamente tambem no servidor

cd ~/Downloads/dinamic-flow-update-for-minifi/nifi/conf/

sudo git clone https://Migueldv06:toekn@github.com/Migueldv06/nifi-teste.git

sudo chown -R 1000:1000 nifi-teste

agora vc pore verificar a atualização do seu projeto na pasta conf/nifi-teste toda vez que você comitar uma atualização do seu flow 
essa pasta não ira interferir no seu servidor de nifi, ela é apenas para o nifi ter o "mesmo ambiente" que os minifis terão

# MiNiFi
baixe o minifi 2.11.0 e descompacte na pasta minifi

cd ~/Downloads/dinamic-flow-update-for-minifi/minifi
wget https://dlcdn.apache.org/nifi/2.11.0/minifi-2.11.0-bin.zip

unzip minifi-2.11.0-bin.zip
rm minifi-2.11.0-bin.zip

edite o arquivo minifi/conf/bootstrap.conf e altere as linhas ou adicione ao final do arquivo:
nifi.minifi.notifier.ingestors=org.apache.nifi.minifi.bootstrap.configuration.ingestors.FileChangeIngestor
nifi.minifi.notifier.ingestors.file.config.path=./conf/nifi-teste/default/flow-minifi.json
nifi.minifi.notifier.ingestors.file.polling.period.seconds=1

nano minifi-2.11.0/conf/bootstrap.conf

faça o clone da imagem do github para dentro da pasta conf/nifi-teste
cd ~/Downloads/dinamic-flow-update-for-minifi/minifi/minifi-2.11.0/conf/
git clone https://Migueldv06:toekn@github.com/Migueldv06/nifi-teste.git

execute o minifi
cd ~/Downloads/dinamic-flow-update-for-minifi/minifi/minifi-2.11.0/bin/
./minifi.sh run

e fique de olho nas logs, no meu caso eu fiz um generate flow file com uma mensagem, ola miguel
a cada minuto ele executa e mostra nas logs

para testar, edite a mensagem no nifi server, commite e veja ela mudar nas logs do minifi