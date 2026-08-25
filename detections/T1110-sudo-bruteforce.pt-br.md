# T1110 - Brute Force // Força Bruta (autenticação sudo)

## MITRE ATT&CK
[T1110 - Brute Force](https://attack.mitre.org/techniques/T1110/)

## Fonte de dado
- Índice: `main`
- Sourcetype: `journald` (journalctl-identifier = sudo)

## Lógica de detecção
Casa com a própria linha de resumo do `sudo` ("N incorrect password
attempts"), emitida uma vez por sessão, já com a contagem final
calculada.

## Query SPL
​```spl
index=main "incorrect password attempts"
| rex field=_raw "(?<user>[\w-]+)\s*:\s*(?<attempts>\d+) incorrect password attempts"
| where attempts >= 3
​```

## Configuração do Alert
- Agendamento: cron `*/15 * * * *`, janela de busca de 15 minutos
- Disparo: Número de Resultados > 0
- Throttling: suprime por 30 minutos, baseado no campo `user`
- Ação: Adiciona aos Alertas Disparados

## Notas de iteração da detecção
A versão inicial casava com mensagens individuais de falha do PAM,
mas o PAM emite formatos inconsistentes entre os tipos de falha,
| rex field=_raw "(?<user>[\w-]+)\s*:\s*(?<attempts>\d+) incorrect password attempts"
| where attempts >= 3
​```
## Status
Testado de ponta a ponta em condições próximas de produção (não só
busca manual): disparou de verdade em 25/08/2026 00:00:02, confirmado
tanto pela interface de Triggered Alerts quanto pelo log interno do
agendador (`index=_internal sourcetype=scheduler`), com
`result_count=1` e o throttling (`suppressed=1`) entrando corretamente
em ação na execução seguinte.

## Limitações conhecidas
- O bloqueio local (`pam_faillock`) já mitiga o ataque; esta detecção
  fornece *visibilidade*, não prevenção.
- Ainda não cobre brute force via SSH, este host não roda `sshd`.