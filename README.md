# Processos, Threads, Escalonamento e Inferência Local com Ollama

Este repositório documenta a instalação, configuração e observação de um sistema local de IA generativa baseado em Ollama. A atividade analisa o impacto da execução do modelo em processos, threads, uso de memória, e chamadas de sistema no sistema operacional Linux.

* **Aluna:** Fernanda Karoliny Santos Silva
* **Instituição:** Universidade Federal de Sergipe (UFS)
* **Curso:** Engenharia de Computação
* **Disciplina:** Sistemas Operacionais

## 💻 Ambiente de Execução e Hardware
Os experimentos práticos e medições foram conduzidos no seguinte ambiente nativo:
* **Sistema Operacional:** Fedora Linux 44 Workstation
* **Kernel:** Linux 7.1.13-200.fc44.x86_64
* **Processador:** Intel(R) Core(TM) i7-8565U CPU @ 1.80GHz (4 núcleos / 8 threads)
* **Memória RAM:** 8 GB
* **Armazenamento:** SSD 500 GB
* **Runtime de Contêiner:** Docker Engine / Containerd
* **Modelo de Linguagem (LLM):** Microsoft Phi-3.5-mini-instruct (3.8B parâmetros, quantização Q4_0, formato GGUF, tamanho aproximado de 2.2 GB)

## Vídeo da atividade
* **Link para o vídeo da apresentação: https://drive.google.com/drive/u/3/folders/1LV4JWaa5eRcxuhCm5eOV6sodakm_YEa-

## 🚀 Instalação, Execução e Reprodução
Para reproduzir o ambiente e a execução local dos experimentos da Trilha A (Chat local: Ollama + Open WebUI), siga os passos abaixo:

### 1. Pré-requisitos
Certifique-se de que o Docker está instalado e ativo em sua distribuição Linux:
```bash
sudo dnf install docker -y
sudo systemctl enable --now docker
```
O ecossistema foi provisionado utilizando uma versão "bundled" (encapsulada), onde tanto a interface (Open WebUI) quanto o motor de inferência (Ollama) rodam no mesmo contêiner Docker.

### 2. Instalação e Execução Inicial
Para provisionar o ambiente, execute o comando abaixo. Esse comando fará o download automático de uma imagem Docker unificada (que já contém o motor Ollama e o Open WebUI instalados juntos) e iniciará o contêiner.

Optou-se por rodar a aplicação em modo CPU, pois o ambiente de testes possui apenas uma placa de vídeo integrada. Assim, garantimos a compatibilidade total e evitamos travamentos por falta de VRAM dedicada.
```bash
docker run -d -p 3000:8080 -v ollama:/root/.ollama -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:ollama
```
URL do Repositório: https://github.com/open-webui/open-webui

### 3. Download e Carregamento do Modelo
Com o contêiner em execução, baixe o modelo Phi-3.5 para o motor interno do Ollama utilizando o utilitário time para mensurar o tempo exato de download:
```bash
time docker exec -it open-webui ollama pull phi3.5:3.8b-mini-instruct-q4_0
```
Link do Modelo utilizado: https://huggingface.co/microsoft/Phi-3.5-mini-instruct

### 4. Comandos de Inventário do Ambiente
Para extrair as informações de hardware e software que constam na especificação do ambiente, os seguintes comandos foram utilizados:
```bash
cat /etc/os-release
uname -r
lscpu
free -h
df -h
nproc
lspci | grep -E "VGA|3D"
```

### 5. Monitoramento de Processos e Threads
Para rastrear a hierarquia (PID e PPID), os estados (STAT) e a quantidade de threads (NLWP) abertas pelo containerd-shim, camada Python e o binário do Ollama durante a inferência, utilize:
```bash
ps -eo pid,ppid,stat,ni,pri,psr,pcpu,pmem,nlwp,comm --sort=-pcpu
```

### 6. Rastreamento de Chamadas de Sistema - Syscalls
Para interceptar e mensurar as requisições de espaço de usuário para o Kernel durante a inferência do modelo (focando no motor ollama), é necessário atrelar o utilitário strace ao Process ID (PID) do Ollama em execução. Primeiro, identifique o PID do Ollama com o comando ps detalhado na seção anterior. Em seguida, siga os passos abaixo: 

Passo 1: Iniciar o monitoramento
Inicie o strace atrelado ao PID do Ollama para gravar as chamadas em segundo plano:
```bash
sudo strace -f -c -p <PID_DO_OLLAMA> -o strace-resumo.txt
```
Atenção: A flag -f é obrigatória para capturar o comportamento das dezenas de threads criadas dinamicamente.

Passo 2: Fazer uma requisição na IA (Gerar Carga de Trabalho)
Com o terminal do strace rodando e escutando o processo, você precisa interagir com a IA para acionar as requisições ao Kernel.

- Abra o seu navegador web e acesse a interface local do Open WebUI através do endereço: http://localhost:3000
- Na interface, certifique-se de selecionar o modelo Phi-3.5-mini-instruct no topo da tela.
- Envie uma requisição no chat (ex: "Explique o que é um Kernel de forma resumida").

Passo 3: Finalizar e ler os resultados
Após receber a resposta da IA, volte ao terminal onde o strace está executando e pressione Ctrl + C para interromper o monitoramento. O utilitário irá compilar e salvar os dados.

Para ler e exibir a tabela de resultados gerada diretamente no terminal, utilize o comando:
```bash
cat strace-resumo.txt
```

### 7. Experimentos de Desempenho e Coleta de Métricas
Para obter dados brutos precisos em nanossegundos (necessários para o cálculo de latência e vazão) e isolar o motor de inferência de possíveis instabilidades da interface web, a coleta de dados foi realizada diretamente através da API REST nativa do Ollama via linha de comando (`curl`).

Para reproduzir os experimentos, é recomendado o uso de **dois terminais** abertos lado a lado:
*   **Terminal 1:** Para o envio da carga de trabalho (requisições).
*   **Terminal 2:** Para o monitoramento em tempo real do uso de CPU e RAM, utilizando o comando:

    ```bash
    docker stats open-webui
    ```

#### Configuração 1: Execução Padrão (Baseline)
Para medir o desempenho em cenário ideal (uma requisição por vez), envie um prompt configurando a saída sem streaming para obter o JSON consolidado no final:
```bash
docker exec -it open-webui curl -s -X POST http://localhost:11434/api/generate -d '{
  "model": "<NOME_DO_MODELO>",
  "prompt": "<SEU_PROMPT_CURTO_OU_LONGO_AQUI>",
  "stream": false
}'
```
#### Configuração 2: Concorrência (Carga Controlada)
Para testar o comportamento do escalonador do Kernel sob estresse, enviamos duas requisições idênticas simultaneamente para o segundo plano (&), salvando as saídas em arquivos distintos e utilizando o wait para sincronizar a conclusão:

```bash
docker exec open-webui curl -s -X POST http://localhost:11434/api/generate -d '{"model": "<NOME_DO_MODELO>", "prompt": "<PROMPT_1>", "stream": false}' > res_concorrente_1.json & \
docker exec open-webui curl -s -X POST http://localhost:11434/api/generate -d '{"model": "<NOME_DO_MODELO>", "prompt": "<PROMPT_2>", "stream": false}' > res_concorrente_2.json & \
wait
echo "Requisições concorrentes concluídas!"
```

Para ler e exibir a tabela de resultados gerada diretamente no terminal, utilize o comando:
```bash
cat res_concorrente_1.json
cat res_concorrente_2.json
```
#### Configuração 3: Ajuste de Execução Local (Restrição de Threads)
Para avaliar o impacto do paralelismo, limitamos artificialmente o motor de inferência a utilizar apenas 2 threads, simulando um ambiente com restrição de hardware:

```bash
docker exec -it open-webui curl -s -X POST http://localhost:11434/api/generate -d '{
  "model": "<NOME_DO_MODELO>",
  "prompt": "<SEU_PROMPT_AQUI>",
  "stream": false,
  "options": {
    "num_thread": 2
  }
}'
```


