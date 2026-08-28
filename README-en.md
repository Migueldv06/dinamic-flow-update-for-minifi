# Dynamic Flow Update for MiNiFi

An environment demonstrating how to keep **MiNiFi** agents automatically updated from a version-controlled flow on **GitHub**, without needing to manually redeploy each agent.

## Pipeline overview

1. You design the flow in **NiFi** (server) and place it under version control pointing to a GitHub repository.
2. A **GitHub Actions workflow** converts the flow (`NiFi-Flow.json`) to the format read by MiNiFi (`flow-minifi.json`) using the **MiNiFi Toolkit**.
3. Each **MiNiFi** agent runs a periodic `git pull` (via an `ExecuteProcess` processor inside the flow itself) and reloads `flow-minifi.json` whenever it changes.

```
dinamic-flow-update-for-minifi/
├── nifi/
│   ├── docker-compose.yaml
│   ├── Dockerfile
│   └── conf/               # generated in the steps below
│       └── nifi-teste/     # clone of the GitHub repository created below
└── minifi/
    └── minifi-2.11.0/
        └── conf/
            └── nifi-teste/ # clone of the GitHub repository created below

```

## Table of Contents

* [1. NiFi (server)](https://www.google.com/search?q=%231-nifi-server)
* [2. GitHub — repository and token](https://www.google.com/search?q=%232-github--repository-and-token)
* [3. Connecting NiFi to GitHub](https://www.google.com/search?q=%233-connecting-nifi-to-github)
* [4. Flow conversion pipeline (GitHub Actions)](https://www.google.com/search?q=%234-flow-conversion-pipeline-github-actions)
* [5. Mirroring the repository inside the NiFi server](https://www.google.com/search?q=%235-mirroring-the-repository-inside-the-nifi-server)
* [6. Creating test flows in NiFi](https://www.google.com/search?q=%236-creating-test-flows-in-nifi)
* [7. MiNiFi (agent)](https://www.google.com/search?q=%237-minifi-agent)
* [8. Testing the complete flow](https://www.google.com/search?q=%238-testing-the-complete-flow)
* [9. Architecture](https://www.google.com/search?q=%239-architecture)

---

## 1. NiFi (server)

### 1.1 Create project folders

```bash
mkdir nifi
mkdir minifi

```

### 1.2 `nifi/docker-compose.yaml`

```yaml
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

```

### 1.3 `nifi/Dockerfile`

A Dockerfile was created to include `git`, which is required for the server to execute the `ExecuteProcess` that runs `git pull` without errors.

```dockerfile
FROM apache/nifi:2.11.0
USER root
RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
USER nifi

```

### 1.4 Create data directories and adjust permissions

These folders are the volumes mounted in `docker-compose.yaml`. They must exist on the host and be owned by UID `1000` (`nifi` user inside the container).

```bash
cd nifi
mkdir conf logs state database_repository flowfile_repository content_repository provenance_repository
sudo chown -R 1000:1000 conf logs state database_repository flowfile_repository content_repository provenance_repository

```

### 1.5 First run — populating the `conf` directory

Since the `./conf` directory is empty, the bind mount `./conf:/opt/nifi/nifi-current/conf` **overrides** the default `conf` folder inside the image (which contains the default `nifi.properties`). Without these files, NiFi fails during startup with an error similar to:

```
nifi  | File [/opt/nifi/nifi-current/conf/nifi.properties] uncommenting [nifi.python.command]
nifi  | sed: can't read /opt/nifi/nifi-current/conf/nifi.properties: No such file or directory
nifi exited with code 2 (restarting)

```

Therefore, on the first startup, `conf` must be manually populated with default files from the image:

1. **Comment out** the `conf` volume line in `docker-compose.yaml`:
```yaml
volumes:
  # - ./conf:/opt/nifi/nifi-current/conf   # comentado só na primeira subida
  - ./logs:/opt/nifi/nifi-current/logs
  # ...demais volumes

```


2. **Start the container** to let NiFi generate the default files internally:
```bash
docker compose up -d

```


3. **Copy the generated files** back to the host:
```bash
sudo docker cp nifi:/opt/nifi/nifi-current/conf/. ./conf/
sudo chown -R 1000:1000 ./conf

```


4. **Uncomment** the `conf` volume line and start again:
```bash
docker compose down
docker compose up -d

```



> 💡 If the same error appears for `state`, `database_repository`, etc., repeat this process for the respective folder.

### 1.6 Accessing NiFi

```
https://localhost:8443/nifi/

```

* **login:** `admin`
* **password:** `sejalivrenifi2026` *(define yours in `docker-compose.yaml`)*

---

## 2. GitHub — repository and token

1. Create a repository on GitHub (can be **private** for security) — e.g.: `nifi-teste`.
2. Add an initial `README.md`.
3. Generate a **Personal Access Token (classic)**:
`Settings > Developer settings > Personal access tokens > Generate new token (classic)`
Grant the **repo** scope (repository access).

> 🔒 Keep your token secure — it will be used to authenticate `git clone`/`git pull` on both NiFi and MiNiFi, as well as in NiFi's Registry Client.

> 🔒 You can use these tokens to restrict access for agents by assigning individual tokens and deleting them if you wish to disable a specific minifi agent.

---

## 3. Connecting NiFi to GitHub

### 3.1 Create the Registry Client

In the top-right menu (3 horizontal bars icon): **Controller Settings → Registry Clients** → add a **`GitHubFlowRegistryClient`** with:

| Field | Value (example) |
| --- | --- |
| Repository Owner | `Migueldv06` |
| Repository Name | `nifi-teste` |
| Authentication Type | Personal Access Token |
| Personal Access Token | `********` |

### 3.2 Add the process group to version control

Right-click your process group → **Version → Start version control**:

* Select the `GitHubFlowRegistryClient` created above.
* Provide a **Flow Name** (e.g.: `NiFi-Flow`).
* Click **Save**.

From this point on, any change to the flow can be committed directly to the repository through NiFi.

---

## 4. Flow conversion pipeline (GitHub Actions)

### 4.1 Clone the repository and download MiNiFi Toolkit

```bash
git clone git@github.com:Migueldv06/nifi-teste.git
cd nifi-teste

wget https://dlcdn.apache.org/nifi/2.11.0/minifi-toolkit-2.11.0-bin.zip
unzip minifi-toolkit-2.11.0-bin.zip
rm minifi-toolkit-2.11.0-bin.zip

```

The **MiNiFi Toolkit** converts the flow exported from NiFi (`NiFi-Flow.json` format) into the format read by MiNiFi agents (`flow-minifi.json`).

### 4.2 Create the workflow

In GitHub, open the **Actions → simple workflow → Configure** tab, and create the `.github/workflows/gerar-flow-minifi.yml` file:

```yaml
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

```

**How it works:** on every `push` to `main` modifying `.json`, `.xml`, or the toolkit itself, it installs Java 21, runs `config.sh transform-nifi` to convert the flow, and commits the resulting `flow-minifi.json` back to the repository.

> ⚠️ Ensure that the paths/filenames used in the workflow (`./default/NiFi-Flow.json`) match the actual files generated by NiFi in your repository.

### 4.3 Push changes and verify

```bash
git pull && git add . && git commit -m "adicionando minifi toolkit" && git push

```

Next, edit your flow in NiFi and click **Save** (commit version). Verify that the file `default/flow-minifi.json` is generated/updated in the repository by the Action.

---

## 5. Mirroring the repository inside the NiFi server

The test flow uses an `ExecuteProcess` processor to run `git pull` inside `./conf/nifi-teste`. This directory must exist and contain the repository clone so the processor does not fail:

```bash
cd dinamic-flow-update-for-minifi/nifi/conf/
sudo git clone https://Migueldv06:<TOKEN>@github.com/Migueldv06/nifi-teste.git nifi-teste
sudo chown -R 1000:1000 nifi-teste

```

> This folder **does not affect** the NiFi server operations — it exists only to provide NiFi with the "same environment" (the same repository layout) that MiNiFi agents will have, allowing you to test `ExecuteProcess` locally before deploying to agents.

---

## 6. Creating test flows in NiFi

### 6.1 Simple test flow

Create a test **process group** containing:

* `GenerateFlowFile` (with test content/attributes)
* connected to a `LogAttribute`

This serves only to generate data and confirm that the versioned flow is being applied correctly.

### 6.2 Dynamic update flow (periodic `git pull`)

Create another flow using an **`ExecuteProcess`** processor:

| Property | Value |
| --- | --- |
| Command | `git` |
| Command Arguments | `pull origin main -q` |
| Working Directory | `./conf/nifi-teste` |
| Scheduler | `1 min` (adjust as needed) |

Connect its output to a `LogAttribute` and auto-terminate/discard the flowfile afterward (it serves strictly as a trigger for `git pull` and carries no payload data).

> If `./conf/nifi-teste` does not exist yet, `ExecuteProcess` will report execution errors — refer to [section 5](https://www.google.com/search?q=%235-mirroring-the-repository-inside-the-nifi-server).

---

## 7. MiNiFi (agent)

### 7.1 Download

```bash
cd ~/Downloads/dinamic-flow-update-for-minifi/minifi
wget https://dlcdn.apache.org/nifi/2.11.0/minifi-2.11.0-bin.zip
unzip minifi-2.11.0-bin.zip
rm minifi-2.11.0-bin.zip

```

### 7.2 `conf/bootstrap.conf`

Edit (or append to the end of) `minifi-2.11.0/conf/bootstrap.conf`:

```bash
nano minifi-2.11.0/conf/bootstrap.conf

```

```properties
nifi.minifi.notifier.ingestors=org.apache.nifi.minifi.bootstrap.configuration.ingestors.FileChangeIngestor
nifi.minifi.notifier.ingestors.file.config.path=./conf/nifi-teste/default/flow-minifi.json
nifi.minifi.notifier.ingestors.file.polling.period.seconds=1

```

| Parameter | Description |
| --- | --- |
| `nifi.minifi.notifier.ingestors` | Class responsible for detecting changes in the flow configuration file. |
| `...ingestors.file.config.path` | Path to `flow-minifi.json` that MiNiFi should monitor and reload when changed. |
| `...file.polling.period.seconds` | Polling interval (in seconds) between checking for file changes. |

### 7.3 Clone the repository inside the agent directory

The path defined above (`./conf/nifi-teste/...`) must physically exist:

```bash
cd ~/Downloads/dinamic-flow-update-for-minifi/minifi/minifi-2.11.0/conf/
git clone https://Migueldv06:<TOKEN>@github.com/Migueldv06/nifi-teste.git nifi-teste

```

### 7.4 Execution

```bash
cd ~/Downloads/dinamic-flow-update-for-minifi/minifi/minifi-2.11.0/bin/
./minifi.sh run

```

Monitor logs — you will see `GenerateFlowFile` executing on each scheduled cycle.

---

## 8. Testing the complete flow

1. In NiFi, edit the message inside `GenerateFlowFile` (e.g.: `"ola miguel"` → `"ola miguel v2"`).
2. Commit the new flow version (**Version → Commit local changes**).
3. Wait for the GitHub Action to run and update `default/flow-minifi.json` in the repository.
4. On the server, `ExecuteProcess` (running every 1 min) fetches updates using `git pull` in `conf/nifi-teste`.
5. On the MiNiFi agent, `FileChangeIngestor` detects modifications in `flow-minifi.json` and reloads the flow automatically.
6. Verify in the MiNiFi logs (`./bin/minifi.sh run`) that the updated message is displayed.

## 9. Architecture

System architecture illustration:

---
