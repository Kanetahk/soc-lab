# SOC Lab

**Bottom line:** um lab de SOC pessoal que leva uma tentativa real de
brute force no `sudo`, do log bruto do `journald` até um alerta T1110
disparando no Splunk, e dali para um pipeline de automação por meio de
um webhook. A detecção foi testada com o alerta disparando de verdade,
não apenas com uma busca manual. O `docker compose up` sobe o Splunk;
o Universal Forwarder e o próprio alerta são configurados à mão (veja
Configuração).

*Read this in: [English](./README.md)*

## Início rápido

O lab inteiro, em ordem. Cada comando `celery` e `uvicorn` vai no seu próprio terminal, iniciado na raiz do repositório.

**Só na primeira vez**, no repositório [splunk-alert-response-pipeline](https://github.com/Kanetahk/splunk-alert-response-pipeline)
(veja a [Configuração](https://github.com/Kanetahk/splunk-alert-response-pipeline/blob/main/locales/pt-BR/README.md#configuração) dele):

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
sudo systemctl enable --now valkey
```

O `.env` precisa estar preenchido antes da primeira execução.

**Toda vez:**

```bash
# 1. soc-lab/ = Splunk
docker compose up -d

# 2. splunk-alert-response-pipeline/ = worker do Celery
source .venv/bin/activate
cd src
celery -A tasks worker --loglevel=info

# 3. splunk-alert-response-pipeline/ = webhook
source .venv/bin/activate
cd src
uvicorn api:app --host 0.0.0.0 --port 8001
```

## Construído com

Splunk Enterprise 9.3 · Docker Compose · systemd-journald · auditd · Splunk Universal Forwarder

## Arquitetura

O Splunk Enterprise roda **containerizado** (Docker Compose), atuando
como Indexer e Search Head. O
**Universal Forwarder roda nativamente no host**, fora de container,
pois ele precisa de acesso direto ao `systemd-journald` do próprio host
para coletar eventos de autenticação, algo que um container não enxerga
sem mounts extras. O Forwarder envia os eventos ao Indexer containerizado
pela porta 9997.

```
Host (systemd-journald, eventos do sudo)
   │
   ▼
Universal Forwarder (nativo)
   │  porta 9997
   ▼
Splunk Indexer / Search Head (container Docker)
   │  busca agendada: T1110, a cada 15 minutos
   ▼
Alerta dispara (3 ou mais tentativas de senha incorreta no sudo)
   │  Ação de Webhook >> http://127.0.0.1:8001/webhook
   ▼
splunk-alert-response-pipeline (FastAPI >> Celery >> DB >> e-mail)
```

## Estrutura do projeto

```
soc-lab/
├── detections/
│   ├── T1110-sudo-bruteforce.md
│   └── T1110-sudo-bruteforce.pt-br.md
├── locales/
│   └── pt-br
│       └── README.md
├── .gitignore
├── docker-compose.yml
└── README.md
```

## Detecções

[T1110 - Brute Force (autenticação via sudo)](../../detections/T1110-sudo-bruteforce.md)
— agendada a cada 15 minutos, dispara quando uma única execução do `sudo`
termina com 3 ou mais tentativas de senha incorreta. Sua ação de alerta
do tipo Webhook é consumida pelo repositório irmão
[splunk-alert-response-pipeline](https://github.com/Kanetahk/splunk-alert-response-pipeline),
que cuida do enriquecimento, persistência e notificação. A query SPL, a
configuração do alerta e as notas da detecção estão no arquivo linkado.

## Configuração

```bash
docker compose up -d
```

Variáveis de ambiente necessárias (`.env`, nunca commitado):
```
SPLUNK_PASSWORD=...
SPLUNK_FWD_PASSWORD=...
```

Splunk Web disponível em `http://127.0.0.1:8000` (`admin` / `SPLUNK_PASSWORD`).

O Universal Forwarder é instalado separadamente, direto no host
(não faz parte dos containers deste repositório), baixe em
[splunk.com](https://www.splunk.com/en_us/download/universal-forwarder.html),
configure o input modular do journald e aponte para
`127.0.0.1:9997`.

## Uso

Confirme que a ingestão está funcionando:
```spl
index=main sourcetype=journald
```

Confirme a lógica de uma detecção específica, veja a query SPL e a
configuração completa do alerta em [detections/T1110-sudo-bruteforce.md](../../detections/T1110-sudo-bruteforce.md).

## Ingestão de log

Configurado um Splunk Universal Forwarder nativo no host, usando o
input modular do journald para coletar eventos de autenticação do `sudo`
e encaminhá-los ao Indexer containerizado pela porta 9997.

**Nota:** o campo COMMAND do sudo captura os argumentos completos da
linha de comando, o que pode incluir dado sensível (ex: senhas passadas
inline). Em um SIEM de produção, isso exigiria mascaramento de campo e
filtragem de dado sensível antes da indexação.

## Limitações conhecidas

- **Versão do Splunk fixada por hardware.** Este host (Intel i3-2328M,
  Sandy Bridge mobile) não tem AES-NI habilitado, a Intel desabilitou
  esse conjunto de instruções em alguns SKUs de i3 mobile dessa
  geração. Versões do Splunk a partir da 9.4/10.x exigem AES-NI/AVX no
  binário do KVStore (MongoDB), sem possibilidade de contornar via
  variável de ambiente. A imagem está fixada em `splunk/splunk:9.3`, a
  última linha de versão ainda compatível com esse hardware.
- **Container roda em `network_mode: host`.** Isso remove o isolamento
  de rede do Docker para o container do Splunk, uma troca deliberada,
  feita depois que a rede em bridge (container >> webhook no host) se
  mostrou pouco confiável em combinação com firewalld/nftables nesse
  host específico. Aceitável para um lab de usuário único; não é um
  padrão para levar a um ambiente compartilhado ou de produção.
- **Sem TLS no Splunk Web nem no link Forwarder-Indexer.** O tráfego
  entre Forwarder e Indexer, e o acesso ao próprio Splunk Web, não são
  criptografados, tranquilo para tráfego de lab restrito a `127.0.0.1`,
  não para qualquer deploy acessível por rede não confiável.
- **Nó único, sem HA.** Um Indexer, um Search Head, no mesmo container,
  suficiente para provar o pipeline de detecção, não dimensionado nem
  arquitetado para volume de log ou requisitos de disponibilidade de
  produção.

O alerta não é criado pelo `docker compose`. Crie-o no Splunk Web a
partir da query SPL e da configuração do alerta em [detections/T1110-sudo-bruteforce.md](../../detections/T1110-sudo-bruteforce.md).

Os dados e a configuração do Splunk (alertas inclusos) ficam em dois
volumes nomeados, `splunk-data` e `splunk-etc`. O `docker compose down`
os mantém; o `docker compose down -v` os apaga, junto com o alerta.

O `auditd` também roda no host, vigiando leituras de `/etc/shadow` e
escritas em `/var/spool/cron` e `/etc/systemd/system`. Um segundo
input do Forwarder (`monitor:///var/log/audit/audit.log`,
`sourcetype=linux_audit`) envia esses eventos ao mesmo Indexer, e o
usuário do Forwarder lê o arquivo por meio de `log_group = splunkread`
no `auditd.conf`. Nenhuma detecção foi construída sobre esses dados
ainda. As regras brutas também casam com processos legítimos
(`unix_chkpwd`, `systemd-userwork`), então um alerta real precisará
filtrar por processo.

## Projetos relacionados

- [splunk-alert-response-pipeline](https://github.com/Kanetahk/splunk-alert-response-pipeline)
  o pipeline de automação que essas detecções alimentam.
- [siem-detection-simulator](https://github.com/Kanetahk/siem-detection-simulator)
  gerador de logs sintéticos para testar essas detecções sem um ataque real