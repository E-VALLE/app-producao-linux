# Prática 05 (desafio opcional): Governança do ambiente

## Atividade 1: Execução e auditoria do script de backup

### Comandos executados

```bash
cd ~/app-producao
mkdir -p backups
cat scripts/manutencao-logs.sh
chmod +x scripts/manutencao-logs.sh
./scripts/manutencao-logs.sh
ls -lh backups/
cat logs/manutencao.log
```

### Saída relevante

```text
total 4,0K
-rw-r--r-- 1 evalle evalle 2,2K Nov 14 09:17 access_20261114_091720.log
```

```text
[sáb. 14 nov 2026 09:17:20 -03] Iniciando rotina de manutencao...
Backup de access.log gerado com sucesso: access_20261114_091720.log
```

## Teste de falha: pasta `backups` sem permissão de escrita

### Comandos executados

```bash
cd ~/app-producao
ls -ld backups
wc -l logs/manutencao.log
chmod 500 backups
ls -ld backups
./scripts/manutencao-logs.sh
echo "codigo de saida: $?"
tail -n 3 logs/manutencao.log
ls -l backups/
chmod 775 backups
ls -ld backups
```

### Saída relevante

```text
drwxrwxr-x 2 evalle evalle 4096 Nov 14 09:17 backups
2 logs/manutencao.log
dr-x------ 2 evalle evalle 4096 Nov 14 09:17 backups
cp: não foi possível criar arquivo comum '/home/evalle/app-producao/backups/access_20261114_093426.log': Permissão negada
codigo de saida: 0
Backup de access.log gerado com sucesso: access_20261114_091720.log
[sáb. 14 nov 2026 09:34:26 -03] Iniciando rotina de manutencao...
Backup de access.log gerado com sucesso: access_20261114_093426.log
total 4
-rw-r--r-- 1 evalle evalle 2222 Nov 14 09:17 access_20261114_091720.log
drwxrwxr-x 2 evalle evalle 4096 Nov 14 09:17 backups
```

## Análise crítica – Script de manutenção e cópia de logs

O script `manutencao-logs.sh` foi desenvolvido para automatizar uma rotina de manutenção, criando uma cópia de segurança especificamente do arquivo `access.log` da aplicação.

A existência do arquivo é verificada pela condição `if [ -f "$LOGS_DIR/access.log" ]`. Esse teste verifica se o caminho informado corresponde a um arquivo existente. Dessa forma, o script somente tenta realizar o backup quando encontra o arquivo, evitando uma tentativa de cópia de um log inexistente.

A identificação dos arquivos de backup é feita pela variável `DATA_HORA`, preenchida pelo comando `date +"%Y%m%d_%H%M%S"`. Essa formatação inclui ano, mês, dia, hora, minuto e segundo. Ao incorporar essa variável ao nome do arquivo, por meio de `access_$DATA_HORA.log`, cada cópia recebe uma identificação própria. Isso facilita a organização e evita que backups de execuções em momentos diferentes sejam sobrescritos. Entretanto, duas execuções dentro do mesmo segundo podem gerar nomes iguais, fazendo com que o backup anterior seja sobrescrito.

O arquivo `manutencao.log` mantém um histórico das execuções do script. O operador `>>`, utilizado com o comando `echo`, acrescenta novas informações sem apagar os registros anteriores. Isso permite acompanhar quando a manutenção foi iniciada e consultar as atividades registradas.

O teste com a pasta `backups` sem permissão de escrita demonstrou uma falha importante. Após executar `chmod 500 backups`, o comando `cp` não conseguiu criar o arquivo de backup e apresentou a mensagem `Permissão negada`. Mesmo assim, o script registrou uma mensagem de sucesso. Ao consultar o histórico com `tail -n 3 logs/manutencao.log`, foi encontrada a linha `Backup de access.log gerado com sucesso: access_20261114_093426.log`. Entretanto, esse arquivo não existia na pasta `backups`, conforme confirmado pelo comando `ls -l backups/`, que mostrou apenas o backup anterior, `access_20261114_091720.log`.

O comando `echo "codigo de saida: $?"` apresentou o valor `0`. Esse valor corresponde ao código de saída do script, e não ao resultado do comando `cp`. Como o último comando executado dentro do script foi o `echo` que registra a mensagem de sucesso, seu retorno foi zero, mesmo após a falha na cópia. Isso demonstra que o script precisa verificar o resultado do comando `cp` antes de registrar uma confirmação de sucesso.

Quando o arquivo `access.log` não existe, o script simplesmente ignora a operação, sem registrar essa ocorrência no `manutencao.log`. Registrar essa informação seria importante para facilitar a identificação de possíveis problemas durante as verificações de manutenção.

Após o teste, a permissão original da pasta `backups` foi restaurada com `chmod 775 backups`.

Os testes demonstraram que o script consegue localizar o arquivo de log, criar uma cópia de segurança identificada pela data e hora e manter um histórico das execuções. Também permitiram identificar, na prática, que uma mensagem de sucesso não significa necessariamente que a operação foi concluída. Para tornar a rotina mais confiável, seria importante verificar o resultado do comando `cp`, registrar quando o arquivo original estiver ausente e melhorar a identificação dos backups para evitar sobrescritas em execuções muito próximas.

Vale destacar que o script ainda realiza apenas uma cópia de segurança, não uma rotação efetiva dos logs, pois não esvazia, renomeia ou substitui o arquivo `access.log` original.
