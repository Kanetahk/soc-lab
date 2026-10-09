# T1110 - Brute Force (autenticação via sudo)

## MITRE ATT&CK
[T1110 - Brute Force](https://attack.mitre.org/techniques/T1110/)

*Read this in: [English](./detections/T1110-sudo-bruteforce.md)*

## Fonte de dados
- Índice: `main`
- Sourcetype: `journald` (journalctl-identifier = sudo)

## Lógica de detecção
Casa com a linha de resumo do próprio sudo ("N incorrect password
attempts"), emitida uma vez por execução do `sudo` com a contagem final
já calculada.

A severidade é calculada na própria busca: `high` com 5 ou mais
tentativas, `medium` caso contrário. O campo viaja até o pipeline
dentro do `result` do payload do webhook.

## Query SPL
```spl
index=main "incorrect password attempts"
| rex field=_raw "(?<user>[\w-]+)\s*:\s*(?<attempts>\d+) incorrect password attempts"
| where attempts >= 3
| eval severity=if(attempts >= 5, "high", "medium")
```

## Configuração do alerta
- Agendamento: cron `*/15 * * * *`, janela de busca de 15 minutos
- Gatilho: Number of Results > 0, disparar para cada resultado
- Throttle: suprimir por 30 minutos, com base em `user`
- Ações: Add to Triggered Alerts, e Webhook para `http://127.0.0.1:8001/webhook`

## Notas de iteração da detecção
A versão inicial casava mensagens individuais do PAM, mas o PAM emite
formatos inconsistentes entre os tipos de falha, causando subcontagem.
Foi revisada para casar a linha de resumo do próprio sudo, mais
confiável, imune à variação de formato das mensagens do PAM.

## Status
Testada de ponta a ponta em condições próximas de produção
(não só busca manual): disparou de verdade em 2026-08-25 00:00:02,
confirmado tanto pela UI de Triggered Alerts quanto pelo log interno do
agendador (`index=_internal sourcetype=scheduler`), com `result_count=1`
e o throttle (`suppressed=1`) atuando corretamente na execução seguinte.

A ação de Webhook foi confirmada disparando de verdade em 2026-09-09
(`fired=1`, `alert_actions="webhook"` no log do agendador), com o
payload completo recebido pelo pipeline.

O campo `severity` e a ramificação do pipeline sobre ele foram
validados apenas com payloads simulados. Com o `passwd_tries` padrão do
sudo, que é 3, espera-se que um alerta real saia como `medium` (veja
Limitações conhecidas).

## Limitações conhecidas
- O bloqueio local do host (`pam_faillock`) já mitiga o ataque;
  essa detecção oferece visibilidade, não prevenção.
- Ainda sem cobertura de brute force por SSH, este host não roda sshd.
- A severidade `high` (5 ou mais tentativas) não deve ser alcançada com
  as configurações padrão do sudo: a linha de resumo conta as tentativas
  dentro de uma única execução do `sudo`, e o `passwd_tries` padrão é 3
  (nenhuma sobrescrita está definida em `/etc/sudoers` ou
  `/etc/sudoers.d` neste host). A ramificação funciona de ponta a ponta
  com payloads simulados, mas um alerta real hoje sai como `medium`.