# SOC Lab

Laboratório pessoal de SOC (Security Operations Center) Splunk, detecção
de ameaças e automação de resposta.

## Limitações conhecidas
Este host (Intel i3-2328M, Sandy Bridge mobile) não possui AES-NI
habilitado, a Intel desabilitou essa instrução em alguns SKUs de i3
mobile dessa geração. Versões do Splunk a partir da 9.4/10.x exigem
AES-NI/AVX no binário do KVStore (MongoDB), sem possibilidade de
contornar via variável de ambiente. A imagem está fixada em
`splunk/splunk:9.3`, a última linha de versão compatível com este hardware.

## Ingestão de log
Configurado um Splunk Universal Forwarder nativo no host, usando o input
modular journald para coletar eventos de autenticação do `sudo` e
encaminhá-los ao Indexer containerizado via porta 9997.

**Nota:** o campo COMMAND do sudo captura os argumentos completos da
linha de comando, o que pode incluir dado sensível (ex: senhas passadas
inline). Em um SIEM de produção, isso exigiria mascaramento de campo/
filtragem de dado sensível antes da indexação.