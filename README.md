# Processos, Threads, Escalonamento e Inferência Local com Ollama

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

## 🚀 Instalação, Execução e Reprodução
Para reproduzir o ambiente e a execução local dos experimentos da Trilha A, siga os passos abaixo:

### 1. Pré-requisitos
Certifique-se de que o Docker está instalado e ativo em sua distribuição Linux:
```bash
sudo dnf install docker -y
sudo systemctl enable --now docker
