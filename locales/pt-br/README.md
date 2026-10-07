# SOC Lab

Laboratório pessoal de SOC (Security Operations Center): Splunk,
detecção de ameaças e automação de resposta.

*Leia isso em: [English](./README.md)*

## Construído com

Splunk Enterprise 9.3 · Docker Compose · systemd-journald · Splunk Universal Forwarder

## Arquitetura

O Splunk Enterprise roda **containerizado** (Docker Compose), atuando
como Indexer e Search Head. O **Universal Forwarder roda nativamente
no host**, fora de container — ele precisa de acesso direto ao
`systemd-journald` do próprio host pra coletar eventos de
autenticação, algo que um container não enxerga sem mounts extras. O
Forwarder envia os eventos pro Indexer containerizado via porta 9997.

```
Host (systemd-journald)
   │
   ▼
Universal Forwarder (nativo)
   │  porta 9997
   ▼
Splunk Indexer/Search Head (container Docker)
```

## Estrutura do projeto

```
├── docker-compose.yml
├── detections/
│   └── T1110-sudo-bruteforce.md
└── .env
```

## Detecções

[T1110 - Brute Force (autenticação via sudo)](./detections/T1110-sudo-bruteforce.md)
— dispara uma ação de alerta do tipo Webhook, consumida pelo
repositório irmão [soar-lab](https://github.com/Kanetahk/soar-lab),
que cuida do enriquecimento, persistência e notificação.

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

O Universal Forwarder é instalado separadamente, direto no host (não
faz parte dos containers desse repositório) — baixa em
[splunk.com](https://www.splunk.com/en_us/download/universal-forwarder.html),
configura o input modular journald, e aponta pra `127.0.0.1:9997`.

## Uso

Confirma que a ingestão está funcionando:
```spl
index=main sourcetype=journald
```

Confirma a lógica de uma detecção específica — veja a query SPL e a
configuração completa do alerta em
[detections/T1110-sudo-bruteforce.md](./detections/T1110-sudo-bruteforce.md).

## Ingestão de log

Configurado um Splunk Universal Forwarder nativo no host, usando o
input modular journald pra coletar eventos de autenticação do `sudo`
e encaminhá-los ao Indexer containerizado via porta 9997.

**Nota:** o campo COMMAND do sudo captura os argumentos completos da
linha de comando, o que pode incluir dado sensível (ex: senhas
passadas inline). Em um SIEM de produção, isso exigiria mascaramento
de campo/filtragem de dado sensível antes da indexação.

## Limitações conhecidas

- **Versão do Splunk fixada por hardware.** Este host (Intel i3-2328M,
  Sandy Bridge mobile) não tem AES-NI habilitado — a Intel desabilitou
  esse conjunto de instruções em alguns SKUs de i3 mobile dessa
  geração. Versões do Splunk a partir da 9.4/10.x exigem AES-NI/AVX no
  binário do KVStore (MongoDB), sem possibilidade de contornar via
  variável de ambiente. A imagem está fixada em `splunk/splunk:9.3`, a
  última linha de versão ainda compatível com esse hardware.
- **Container roda em `network_mode: host`.** Isso remove o isolamento
  de rede do Docker pro container do Splunk — uma troca deliberada,
  feita depois que a rede em bridge (container → webhook no host) se
  mostrou pouco confiável em combinação com firewalld/nftables nesse
  host específico. Aceitável pra um lab de usuário único; não é um
  padrão pra levar pra um ambiente compartilhado ou de produção.
- **Sem TLS no Splunk Web nem no link Forwarder-Indexer.** O tráfego
  entre Forwarder e Indexer, e o acesso ao próprio Splunk Web, não são
  criptografados — tranquilo pra tráfego de lab restrito a
  `127.0.0.1`, não pra qualquer deploy acessível por rede não
  confiável.
- **Nó único, sem HA.** Um Indexer, um Search Head, no mesmo
  container — suficiente pra provar o pipeline de detecção, não
  dimensionado nem arquitetado pra volume de log ou requisitos de
  disponibilidade de produção.