Price Prediction Worker Node
System Requirements
image

Install dependecies
# Install Packages
sudo apt update & sudo apt upgrade -y

sudo apt install ca-certificates zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev curl git wget make jq build-essential pkg-config lsb-release libssl-dev libreadline-dev libffi-dev gcc screen unzip lz4 -y
# Install Python3
sudo apt install python3
python3 --version

sudo apt install python3-pip
pip3 --version
# Install Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
docker version

# Install Docker-Compose
VER=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep tag_name | cut -d '"' -f 4)

curl -L "https://github.com/docker/compose/releases/download/"$VER"/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose
docker-compose --version

# Docker Permission to user
sudo groupadd docker
sudo usermod -aG docker $USER
# Install Go
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.4.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> $HOME/.bash_profile
source .bash_profile
go version
Install Allorad: Wallet
git clone https://github.com/allora-network/allora-chain.git

cd allora-chain && make all

allorad version
Add Wallet
You can use your keplr seed-phrase to recover your wallet or create a new one
# Recover your wallet with seed-phrase
allorad keys add testkey --recover

#OR

# Create a new wallet
allorad keys add testkey
Get Faucet
Connect to Allora dashboard to find your Allora address

You can add Allora network to Keplr here

Get uAllo faucet here

Screenshot_77

Screenshot_76

Install Worker
# Install
cd $HOME && git clone https://github.com/allora-network/basic-coin-prediction-node

cd basic-coin-prediction-node

mkdir workers
mkdir workers/worker-1 workers/worker-2 head-data

# Give certain permissions
sudo chmod -R 777 workers/worker-1
sudo chmod -R 777 workers/worker-2
sudo chmod -R 777 head-data

# Create head keys
sudo docker run -it --entrypoint=bash -v ./head-data:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"

# Create worker keys
sudo docker run -it --entrypoint=bash -v ./workers/worker-1:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"
sudo docker run -it --entrypoint=bash -v ./workers/worker-2:/data alloranetwork/allora-inference-base:latest -c "mkdir -p /data/keys && (cd /data/keys && allora-keys)"
# Copy the head-id
cat head-data/keys/identity
This is your head-id , you need it in the next step

Screenshot_78

Connect to Allora Chain
Delete and create new docker-compose.yml file
rm -rf docker-compose.yml && nano docker-compose.yml
Copy & Paste the following code in it
Replace head-id & WALLET_SEED_PHRASE in worker-1 and worker-2 containers
version: '3'

services:
  inference:
    container_name: inference
    build:
      context: .
    command: python -u /app/app.py
    ports:
      - "8000:8000"
    networks:
      eth-model-local:
        aliases:
          - inference
        ipv4_address: 172.22.0.4
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/inference/ETH"]
      interval: 10s
      timeout: 10s
      retries: 12
    volumes:
      - ./inference-data:/app/data
  
  updater:
    container_name: updater
    build: .
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
    command: >
      sh -c "
      while true; do
        python -u /app/update_app.py;
        sleep 24h;
      done
      "
    depends_on:
      inference:
        condition: service_healthy
    networks:
      eth-model-local:
        aliases:
          - updater
        ipv4_address: 172.22.0.5
  
  head:
    container_name: head
    image: alloranetwork/allora-inference-base-head:latest
    environment:
      - HOME=/data
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        allora-node --role=head --peer-db=/data/peerdb --function-db=/data/function-db  \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9010 --rest-api=:6000 \
          --boot-nodes=/dns/head-0-p2p.testnet-1.testnet.allora.network/tcp/32130/p2p/12D3KooWLBhsSucVVcyVCaM9pvK8E7tWBM9L19s7XQHqqejyqgEC,/dns/head-1-p2p.testnet-1.testnet.allora.network/tcp/32131/p2p/12D3KooWEUNWg7YHeeCtH88ju63RBfY5hbdv9hpv84ffEZpbJszt,/dns/head-2-p2p.testnet-1.testnet.allora.network/tcp/32132/p2p/12D3KooWATfUSo95wtZseHbogpckuFeSvpL4yks6XtvrjVHcCCXk,/dns/head-5-p2p.testnet-1.testnet.allora.network/tcp/32135/p2p/12D3KooWAazxKoYszYt4XhCrGWoEUyAFMaU7DB9RZ8TsA7qwLfin,/dns/head-4-p2p.testnet-1.testnet.allora.network/tcp/32134/p2p/12D3KooWRF8HNU21AukE7KC6kZqxqvCiZ5nM9xcLW4YvsuGAYbcm,/dns/head-3-p2p.testnet-1.testnet.allora.network/tcp/32133/p2p/12D3KooWDrArwBSCNxwL3mgJ2NaUygdtPtiwVQtPJafyAH6FSiUf
    ports:
      - "6000:6000"
    volumes:
      - ./head-data:/data
    working_dir: /data
    networks:
      eth-model-local:
        aliases:
          - head
        ipv4_address: 172.22.0.100

  worker-1:
    container_name: worker-1
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
      - HOME=/data
    build:
      context: .
      dockerfile: Dockerfile_b7s
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        # Change boot-nodes below to the key advertised by your head
        allora-node --role=worker --peer-db=/data/peerdb --function-db=/data/function-db \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9011 \
          --boot-nodes=/dns/head-0-p2p.testnet-1.testnet.allora.network/tcp/32130/p2p/12D3KooWLBhsSucVVcyVCaM9pvK8E7tWBM9L19s7XQHqqejyqgEC,/dns/head-1-p2p.testnet-1.testnet.allora.network/tcp/32131/p2p/12D3KooWEUNWg7YHeeCtH88ju63RBfY5hbdv9hpv84ffEZpbJszt,/dns/head-2-p2p.testnet-1.testnet.allora.network/tcp/32132/p2p/12D3KooWATfUSo95wtZseHbogpckuFeSvpL4yks6XtvrjVHcCCXk,/dns/head-5-p2p.testnet-1.testnet.allora.network/tcp/32135/p2p/12D3KooWAazxKoYszYt4XhCrGWoEUyAFMaU7DB9RZ8TsA7qwLfin,/dns/head-4-p2p.testnet-1.testnet.allora.network/tcp/32134/p2p/12D3KooWRF8HNU21AukE7KC6kZqxqvCiZ5nM9xcLW4YvsuGAYbcm,/dns/head-3-p2p.testnet-1.testnet.allora.network/tcp/32133/p2p/12D3KooWDrArwBSCNxwL3mgJ2NaUygdtPtiwVQtPJafyAH6FSiUf,/ip4/172.22.0.100/tcp/9010/p2p/head-id \
          --topic=allora-topic-1-worker --allora-chain-worker-mode=worker \
          --allora-chain-restore-mnemonic='WALLET_SEED_PHRASE' \
          --allora-node-rpc-address=https://allora-rpc.testnet-1.testnet.allora.network \
          --allora-chain-key-name=worker-1 \
          --allora-chain-topic-id=1
    volumes:
      - ./workers/worker-1:/data
    working_dir: /data
    depends_on:
      - inference
      - head
    networks:
      eth-model-local:
        aliases:
          - worker1
        ipv4_address: 172.22.0.12

  worker-2:
    container_name: worker-2
    environment:
      - INFERENCE_API_ADDRESS=http://inference:8000
      - HOME=/data
    build:
      context: .
      dockerfile: Dockerfile_b7s
    entrypoint:
      - "/bin/bash"
      - "-c"
      - |
        if [ ! -f /data/keys/priv.bin ]; then
          echo "Generating new private keys..."
          mkdir -p /data/keys
          cd /data/keys
          allora-keys
        fi
        # Change boot-nodes below to the key advertised by your head
        allora-node --role=worker --peer-db=/data/peerdb --function-db=/data/function-db \
          --runtime-path=/app/runtime --runtime-cli=bls-runtime --workspace=/data/workspace \
          --private-key=/data/keys/priv.bin --log-level=debug --port=9013 \
          --boot-nodes=/dns/head-0-p2p.testnet-1.testnet.allora.network/tcp/32130/p2p/12D3KooWLBhsSucVVcyVCaM9pvK8E7tWBM9L19s7XQHqqejyqgEC,/dns/head-1-p2p.testnet-1.testnet.allora.network/tcp/32131/p2p/12D3KooWEUNWg7YHeeCtH88ju63RBfY5hbdv9hpv84ffEZpbJszt,/dns/head-2-p2p.testnet-1.testnet.allora.network/tcp/32132/p2p/12D3KooWATfUSo95wtZseHbogpckuFeSvpL4yks6XtvrjVHcCCXk,/dns/head-5-p2p.testnet-1.testnet.allora.network/tcp/32135/p2p/12D3KooWAazxKoYszYt4XhCrGWoEUyAFMaU7DB9RZ8TsA7qwLfin,/dns/head-4-p2p.testnet-1.testnet.allora.network/tcp/32134/p2p/12D3KooWRF8HNU21AukE7KC6kZqxqvCiZ5nM9xcLW4YvsuGAYbcm,/dns/head-3-p2p.testnet-1.testnet.allora.network/tcp/32133/p2p/12D3KooWDrArwBSCNxwL3mgJ2NaUygdtPtiwVQtPJafyAH6FSiUf,/ip4/172.22.0.100/tcp/9010/p2p/head-id \
          --topic=allora-topic-2-worker --allora-chain-worker-mode=worker \
          --allora-chain-restore-mnemonic='WALLET_SEED_PHRASE' \
          --allora-node-rpc-address=https://allora-rpc.testnet-1.testnet.allora.network \
          --allora-chain-key-name=worker-2 \
          --allora-chain-topic-id=2
    volumes:
      - ./workers/worker-2:/data
    working_dir: /data
    depends_on:
      - inference
      - head
    networks:
      eth-model-local:
        aliases:
          - worker1
        ipv4_address: 172.22.0.13
  
networks:
  eth-model-local:
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/24

volumes:
  inference-data:
  workers:
  head-data:
To save: CTRL+X+Y Enter

Run worker
docker compose up -d --build
Check your node status
Check running docker containers
# Ensure you are in the right directory
cd $HOME && cd basic-coin-prediction-node

# Check worker 1 logs
docker compose logs -f worker-1

# Check worker 2 logs
docker compose logs -f worker-2
You must have Success: register node Tx Hash in workers 1 & 2 logs Success: register node Tx Hash:=82BF67E2E1247B226B8C5CFCF3E4F41076909ADABF3852C468D087D94BD9FC3B

Screenshot_80

Check Worker node:
Check topic 1:

network_height=$(curl -s -X 'GET' 'https://allora-rpc.testnet-1.testnet.allora.network/abci_info?' -H 'accept: application/json' | jq -r .result.response.last_block_height) && \
curl --location 'http://localhost:6000/api/v1/functions/execute' --header 'Content-Type: application/json' --data '{
    "function_id": "bafybeigpiwl3o73zvvl6dxdqu7zqcub5mhg65jiky2xqb4rdhfmikswzqm",
    "method": "allora-inference-function.wasm",
    "parameters": null,
    "topic": "allora-topic-1-worker",
    "config": {
        "env_vars": [
            {
                "name": "BLS_REQUEST_PATH",
                "value": "/api"
            },
            {
                "name": "ALLORA_ARG_PARAMS",
                "value": "ETH"
            },
            {
                "name": "ALLORA_BLOCK_HEIGHT_CURRENT",
                "value": "'"${network_height}"'"
            }
        ],
        "number_of_nodes": -1,
        "timeout": 2
    }
}' | jq
Check topic 2:

network_height=$(curl -s -X 'GET' 'https://allora-rpc.testnet-1.testnet.allora.network/abci_info?' -H 'accept: application/json' | jq -r .result.response.last_block_height) && \
curl --location 'http://localhost:6000/api/v1/functions/execute' --header 'Content-Type: application/json' --data '{
    "function_id": "bafybeigpiwl3o73zvvl6dxdqu7zqcub5mhg65jiky2xqb4rdhfmikswzqm",
    "method": "allora-inference-function.wasm",
    "parameters": null,
    "topic": "allora-topic-2-worker",
    "config": {
        "env_vars": [
            {
                "name": "BLS_REQUEST_PATH",
                "value": "/api"
            },
            {
                "name": "ALLORA_ARG_PARAMS",
                "value": "ETH"
            },
            {
                "name": "ALLORA_BLOCK_HEIGHT_CURRENT",
                "value": "'"${network_height}"'"
            }
        ],
        "number_of_nodes": -1,
        "timeout": 2
    }
}' | jq
Response: you will get code: 200 if everything is fine

{
  "code": "200",
  "request_id": "9660af22-54d0-4219-a1de-3677868b715f",
  "results": [
    {
      "result": {
        "stdout": "Error running script: exit status 1\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWFqPD3xcY37R5kgUePW4ZNgM2ePDv9r96qF68kwFQZ3pU",
        "12D3KooWJiRzp3DFSy6yqspp4KdJSyDkjBdBfq2a1uf2BRBCdAGk",
        "12D3KooWGdo7GZgkcY7stYQbRYoSMuiSeBoUBiLd8JzUcgfDQCMN"
      ],
      "frequency": 21.428571428571427
    },
    {
      "result": {
        "stdout": "{\"infererValue\": \"3392.0363009974553\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWHRyWdYQhpgrQ2q6CArP8N6UsDXt5mDfML4fybqCFVGEC",
        "12D3KooWPTsR5JMmjtekPzUr9gRdXNit4CMXqDxDHHc2beXcuHyr"
      ],
      "frequency": 14.285714285714286
    },
    {
      "result": {
        "stdout": "{\"infererValue\": \"3473.133811362153\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWKTidHUzFchp1C38WvVEhBAudBCTxiArbxphgyQi2Qvih"
      ],
      "frequency": 7.142857142857143
    },
    {
      "result": {
        "stdout": "{\"infererValue\": \"3426.4731836502633\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWGMAHnaGgoaUz9KiKut9xLEPk66aCW15PLrpotBWS84JM"
      ],
      "frequency": 7.142857142857143
    },
    {
      "result": {
        "stdout": "{\"infererValue\": \"3464.5369537965817\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWJcFcL9arJWbwJKyZpZn81J6Tj5fpFW4BG4AvLeGTTSpi"
      ],
      "frequency": 7.142857142857143
    },
    {
      "result": {
        "stdout": "{\"infererValue\": \"3366.2086390078493\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWFYAveN1qCyyrrkdUDuueAeA5Dyk7GEZyhYnmxhe5arfT"
      ],
      "frequency": 7.142857142857143
    },
    {
      "result": {
        "stdout": "{\"infererValue\": \"3374.8178596710513\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWQMjk6RBAUdsexGoSrGiBRYhEjk6b6QAqiy28uWssJerN"
      ],
      "frequency": 7.142857142857143
    },
    {
      "result": {
        "stdout": "{\"infererValue\": \"3455.94009623101\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWQ85FMueZ6rv6XYHK6FbDqKs7wzZP1kkdqvNVSqo3gA1Z"
      ],
      "frequency": 7.142857142857143
    },
    {
      "result": {
        "stdout": "{\"infererValue\": \"3383.4270803342533\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWFMUvLousm8qm7RDgxMuA9QhgirsvC24vGNU4wFzgCrTD"
      ],
      "frequency": 7.142857142857143
    },
    {
      "result": {
        "stdout": "{\"infererValue\": \"2925.5640664690764\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWKMWhzRkyBZ4YAUSRGgauHd8yi5DQcfHz3h5zZ9Yktdtz"
      ],
      "frequency": 7.142857142857143
    },
    {
      "result": {
        "stdout": "{\"infererValue\": \"3021.1331\"}\n\n",
        "stderr": "",
        "exit_code": 0
      },
      "peers": [
        "12D3KooWMUaYAnYaFWBjHVTxwgYBdnG5P2doKetE4NzypUjG9Ddm"
      ],
      "frequency": 7.142857142857143
    }
  ],
  "cluster": {
    "peers": [
      "12D3KooWFqPD3xcY37R5kgUePW4ZNgM2ePDv9r96qF68kwFQZ3pU",
      "12D3KooWPTsR5JMmjtekPzUr9gRdXNit4CMXqDxDHHc2beXcuHyr",
      "12D3KooWKMWhzRkyBZ4YAUSRGgauHd8yi5DQcfHz3h5zZ9Yktdtz",
      "12D3KooWJcFcL9arJWbwJKyZpZn81J6Tj5fpFW4BG4AvLeGTTSpi",
      "12D3KooWGdo7GZgkcY7stYQbRYoSMuiSeBoUBiLd8JzUcgfDQCMN",
      "12D3KooWQ85FMueZ6rv6XYHK6FbDqKs7wzZP1kkdqvNVSqo3gA1Z",
      "12D3KooWFMUvLousm8qm7RDgxMuA9QhgirsvC24vGNU4wFzgCrTD",
      "12D3KooWQMjk6RBAUdsexGoSrGiBRYhEjk6b6QAqiy28uWssJerN",
      "12D3KooWHRyWdYQhpgrQ2q6CArP8N6UsDXt5mDfML4fybqCFVGEC",
      "12D3KooWJiRzp3DFSy6yqspp4KdJSyDkjBdBfq2a1uf2BRBCdAGk",
      "12D3KooWFYAveN1qCyyrrkdUDuueAeA5Dyk7GEZyhYnmxhe5arfT",
      "12D3KooWKTidHUzFchp1C38WvVEhBAudBCTxiArbxphgyQi2Qvih",
      "12D3KooWGMAHnaGgoaUz9KiKut9xLEPk66aCW15PLrpotBWS84JM",
      "12D3KooWMUaYAnYaFWBjHVTxwgYBdnG5P2doKetE4NzypUjG9Ddm"
    ]
  }
}
Check Updater node:
curl http://localhost:8000/update
Response:

0
Check Inference node:
curl http://localhost:8000/inference/ETH
Response:

{"value":"2564.021586281073"}
Check Docker containers
docker ps
Congratz. Worker is working safely but currently I am not sure that I have done it completely correct as the point system seems to be buggy. I'll update here and on my twitter 0xMoei

🚨Error 408: when checking topic status
# Ensure you are in the right directory
cd $HOME && cd basic-coin-prediction-node

# Remove worker container (worker-1 or worker-2)
docker container stop worker-1
docker container rm worker-1

# Restart worker container (worker-1 or worker-2)
docker compose up -d --build
