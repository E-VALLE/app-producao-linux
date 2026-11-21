# Prática 05 (desafio opcional): Automação e Systemd

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

---

## Atividade 2: Agendamento automático com o Cron

O cron é um serviço do Linux utilizado para automatizar a execução de tarefas em horários programados. Por meio dele, é possível configurar a execução periódica de scripts sem precisar iniciá-los manualmente.

Para cadastrar a rotina de manutenção, foi utilizado o comando:

```bash
crontab -e
```

Esse comando abre o editor da tabela de agendamentos do usuário. A seguinte linha foi adicionada:

```bash
0 0 * * * $HOME/app-producao/scripts/manutencao-logs.sh
```

Essa configuração determina que o script `manutencao-logs.sh` seja executado diariamente à meia-noite.

### Significado dos cinco campos do Cron

A sintaxe do Cron utiliza cinco campos, separados por espaços, que determinam quando uma tarefa será executada.

| Campo | Posição | Significado |
|---|---|---|
| Minuto | 1º | Define o minuto da execução (0 a 59). |
| Hora | 2º | Define a hora da execução (0 a 23). |
| Dia do mês | 3º | Define o dia do mês (1 a 31). |
| Mês | 4º | Define o mês (1 a 12). |
| Dia da semana | 5º | Define o dia da semana (0 a 7), sendo domingo 0 ou 7. |

O caractere `*` representa todos os valores possíveis daquele campo.

### Execução a cada 15 minutos em dias úteis

Para executar o mesmo script a cada 15 minutos, de segunda a sexta-feira, a expressão Cron é:

```bash
*/15 * * * 1-5 $HOME/app-producao/scripts/manutencao-logs.sh
```

Nessa configuração:

- `*/15`: executa a cada 15 minutos (00, 15, 30 e 45).
- `*`: permite a execução em todas as horas.
- `*`: permite a execução em todos os dias do mês.
- `*`: permite a execução em todos os meses.
- `1-5`: restringe a execução de segunda a sexta-feira.

### Verificação do agendamento e da execução

Após salvar a configuração no editor, o comando abaixo permite consultar os agendamentos ativos do usuário:

```bash
crontab -l
```

Esse comando confirma se a expressão foi salva corretamente, mas não comprova que o script foi executado.

Para verificar as execuções, é possível consultar o histórico mantido pelo próprio script:

```bash
tail -n 10 logs/manutencao.log
```

Outra possibilidade é consultar os registros do serviço Cron no sistema:

```bash
grep "(evalle) CMD" /var/log/syslog | tail -n 5
```

Para realizar um teste rápido, também seria possível alterar temporariamente a programação para executar a cada minuto:

```bash
* * * * * $HOME/app-producao/scripts/manutencao-logs.sh
```

Após aguardar a execução, basta consultar o `manutencao.log` e verificar se foram acrescentados novos registros. Depois do teste, a programação original deve ser restaurada.

### Saída relevante do teste

```text
2026-11-14T10:21:01.907919-03:00 pos-linux-vm CRON[68307]: (evalle) CMD ($HOME/app-producao/scripts/manutencao-logs.sh)
2026-11-14T10:22:01.977234-03:00 pos-linux-vm CRON[68328]: (evalle) CMD ($HOME/app-producao/scripts/manutencao-logs.sh)
2026-11-14T10:23:01.041391-03:00 pos-linux-vm CRON[68355]: (evalle) CMD ($HOME/app-producao/scripts/manutencao-logs.sh)
[sáb. 14 nov 2026 10:21:01 -03] Iniciando rotina de manutencao...
Backup de access.log gerado com sucesso: access_20261114_102101.log
[sáb. 14 nov 2026 10:22:02 -03] Iniciando rotina de manutencao...
Backup de access.log gerado com sucesso: access_20261114_102202.log
[sáb. 14 nov 2026 10:23:01 -03] Iniciando rotina de manutencao...
Backup de access.log gerado com sucesso: access_20261114_102301.log
```

## Análise crítica – Agendamento automático com o Cron

O agendamento automático permite que o backup do arquivo `access.log` seja realizado sem depender de uma execução manual. Na configuração principal, o script está programado para executar diariamente à meia-noite. Para verificar seu funcionamento, foi realizado um teste temporário utilizando a expressão `* * * * *`, que determina a execução a cada minuto.

O teste indicou que o agendamento está funcionando. A consulta ao arquivo `/var/log/syslog`, por meio do comando `grep "(evalle) CMD" /var/log/syslog | tail -n 5`, apresentou registros de três execuções, realizadas às 10:21, 10:22 e 10:23. O histórico `manutencao.log` também registrou essas execuções e a criação dos arquivos `access_20261114_102101.log`, `access_20261114_102202.log` e `access_20261114_102301.log`.

Após a verificação, os três arquivos gerados durante o teste foram removidos com o comando `rm -i`, preservando o backup original `access_20261114_091720.log`. A limpeza evitou manter arquivos desnecessários na pasta `backups`.

A programação original foi restaurada para execução diária à meia-noite, conforme confirmado pelo comando `crontab -l`.

A expressão alternativa `*/15 * * * 1-5` permitiria executar o script a cada 15 minutos, de segunda a sexta-feira. Nesse cenário, seriam realizadas 96 execuções em cada dia útil. Essa frequência aumentaria a quantidade de cópias disponíveis, mas também exigiria atenção ao espaço em disco.

Como o script não possui uma rotina de exclusão de backups antigos, os arquivos permaneceriam armazenados indefinidamente, fazendo com que a pasta `backups` crescesse continuamente. Por esse motivo, seria importante implementar uma política de retenção, definindo por quanto tempo os backups devem ser mantidos e quando os arquivos antigos poderão ser removidos.

Outra limitação da configuração atual é que a linha cadastrada na crontab não possui redirecionamento da saída do script. Dessa forma, mensagens e erros não são armazenados automaticamente em um arquivo específico pelo Cron. Caso ocorra uma falha antes de o script registrar alguma informação no `manutencao.log`, poderá não existir um registro local dessa ocorrência. Uma melhoria seria utilizar o redirecionamento `>>` para acrescentar as mensagens a um arquivo de execução e `2>&1` para direcionar também os erros.

Por exemplo:

```bash
0 0 * * * $HOME/app-producao/scripts/manutencao-logs.sh >> $HOME/app-producao/logs/cron.log 2>&1
```

O teste mostrou a importância de verificar separadamente o agendamento e a execução efetiva. O comando `crontab -l` confirma a programação cadastrada, enquanto os registros do Syslog e do `manutencao.log` permitem acompanhar as execuções realizadas. O redirecionamento proposto acrescentaria uma camada de registro, facilitando a identificação de falhas futuras.

### Situação dos testes

| Verificação | Resultado |
|---|---|
| Agendamento diário à meia-noite | Restaurado |
| Teste de execução a cada minuto | Realizado |
| Registros no Syslog | Confirmados |
| Registros no `manutencao.log` | Confirmados |
| Backups temporários | Removidos |
| Backup original | Preservado |
| Política de retenção | Não implementada |
| Redirecionamento de erros do Cron | Melhoria proposta |

---

## Atividade 3: Unidade de serviço no Systemd

### 1. Criação do arquivo de unidade

Foi criado o arquivo `/etc/systemd/system/app-producao.service` para permitir que o Systemd gerencie a execução da aplicação.

Antes da configuração, foram conferidos o usuário de execução, o diretório da aplicação e o caminho do interpretador Python. Os resultados foram:

- Usuário: `evalle`;
- Diretório de trabalho: `/home/evalle/app-producao`;
- Executável Python: `/usr/bin/python3`;
- Diretório servido: `/home/evalle/app-producao/backend`.

O conteúdo completo do arquivo criado foi:

```ini
[Unit]
Description=Servico da Aplicacao Producao
After=network.target

[Service]
Type=simple
User=evalle
WorkingDirectory=/home/evalle/app-producao
ExecStart=/usr/bin/python3 -m http.server 8080 --directory backend
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

A configuração utiliza o usuário `evalle`, evitando executar a aplicação com privilégios administrativos. A diretiva `WorkingDirectory` define o diretório de trabalho, enquanto `ExecStart` informa o comando que inicia o servidor HTTP do Python na porta 8080, disponibilizando o conteúdo da pasta `backend`.

A diretiva `After=network.target` estabelece a ordem de inicialização em relação ao target de rede. Ela não garante, por si só, que a conexão com a internet esteja disponível, mas determina que essa etapa seja considerada antes da inicialização do serviço.

### 2. Comandos utilizados

Após criar e conferir o arquivo, foram executados os seguintes comandos.

Recarregar as configurações do Systemd:

```bash
sudo systemctl daemon-reload
```

Esse comando atualiza as configurações carregadas pelo Systemd para reconhecer a nova unidade de serviço.

Habilitar a inicialização automática e iniciar o serviço:

```bash
sudo systemctl enable --now app-producao.service
```

O comando habilita a inicialização automática junto com o sistema e inicia o serviço imediatamente.

Consultar o estado do serviço:

```bash
systemctl status app-producao.service
```

O resultado apresentou:

```text
Active: active (running) since Sat 2026-11-14 10:44:46 -03; 28s ago
Main PID: 68993 (python3)
```

Isso confirmou que a aplicação estava em execução e sendo monitorada pelo Systemd.

Verificar a inicialização automática:

```bash
systemctl is-enabled app-producao.service
```

O retorno foi `enabled`, o que confirmou que o serviço estava habilitado para iniciar automaticamente com o sistema.

Consultar os registros de execução:

```bash
sudo journalctl -u app-producao -n 30 --no-pager
```

O Journal apresentou o registro de inicialização do serviço, permitindo acompanhar os eventos relacionados à unidade.

### 3. Teste de recuperação automática

Para verificar o funcionamento da diretiva `Restart=always`, foi realizado um teste de encerramento forçado do processo:

```bash
sudo systemctl kill --signal=SIGKILL app-producao.service
```

Em seguida, foi aguardado um intervalo de sete segundos:

```bash
sleep 7
```

Depois, o estado do serviço foi consultado novamente:

```bash
systemctl status app-producao.service
```

O resultado mostrou que o serviço estava novamente em execução, mas com um novo identificador de processo:

```text
Active: active (running) since Sat 2026-11-14 10:47:13 -03; 19s ago
Main PID: 69040 (python3)
```

Antes do teste, o processo principal era o PID 68993. Após o encerramento forçado, o Systemd iniciou um novo processo, identificado pelo PID 69040.

A consulta aos registros confirmou o que aconteceu:

```bash
sudo journalctl -u app-producao -n 30 --no-pager
```

```text
nov 14 10:44:46 pos-linux-vm systemd[1]: Started app-producao.service - Servico da Aplicacao Producao.
nov 14 10:47:08 pos-linux-vm systemd[1]: app-producao.service: Sent signal SIGKILL to main process 68993 (python3) on client request.
nov 14 10:47:08 pos-linux-vm systemd[1]: app-producao.service: Killed unit cgroup '/system.slice/app-producao.service' with SIGKILL on client request.
nov 14 10:47:08 pos-linux-vm systemd[1]: app-producao.service: Main process exited, code=killed, status=9/KILL
nov 14 10:47:08 pos-linux-vm systemd[1]: app-producao.service: Failed with result 'signal'.
nov 14 10:47:13 pos-linux-vm systemd[1]: app-producao.service: Scheduled restart job, restart counter is at 1.
nov 14 10:47:13 pos-linux-vm systemd[1]: Started app-producao.service - Servico da Aplicacao Producao.
```

Os registros demonstraram que o Systemd identificou o encerramento do processo, programou uma nova tentativa de inicialização e colocou o serviço novamente em execução após cinco segundos.

### 4. Análise crítica – Recuperação automática com Systemd

A utilização do Systemd trouxe mais segurança e controle para a execução da aplicação. Em vez de iniciar o processo manualmente no terminal e mantê-lo em segundo plano, o serviço passou a ser gerenciado diretamente pelo sistema operacional.

É importante esclarecer que, nesta atividade, foi utilizado o servidor HTTP nativo do Python (`http.server`) como substituto para simular a disponibilização de arquivos pelo backend. Ele serve o conteúdo da pasta `backend` na porta 8080, mas não representa a API real do projeto nem implementa suas funcionalidades.

A diretiva `Restart=always` foi fundamental para o teste de recuperação. Ela determina que o Systemd tente reiniciar o processo sempre que ele terminar, inclusive quando a aplicação encerrar normalmente, retornando código de saída zero. Uma exceção importante ocorre quando o serviço é parado explicitamente pelo administrador, por exemplo, com `systemctl stop`: nesse caso, o Systemd respeita a solicitação e não reinicia o serviço.

Existe também a alternativa `Restart=on-failure`, que reinicia o serviço quando ocorre uma falha, como um encerramento por `SIGKILL`, mas não quando o processo termina normalmente com código de saída zero. Portanto, a escolha de `Restart=always` é adequada quando se deseja manter o serviço em execução continuamente, independentemente do motivo pelo qual o processo terminou.

Já a diretiva `RestartSec=5` estabelece um intervalo de cinco segundos antes de cada tentativa de reinicialização. Esse tempo evita tentativas imediatas e sucessivas, ajudando a reduzir a possibilidade de atingir o limite de inicializações do Systemd. Por padrão, esse limite é de cinco tentativas em dez segundos (`StartLimitBurst=5` e `StartLimitIntervalSec=10s`). Quando ultrapassado, o serviço deixa de ser reiniciado automaticamente. Para permitir novas tentativas, é possível zerar o estado de falha com:

```bash
sudo systemctl reset-failed app-producao.service
```

Outra possibilidade é solicitar uma nova inicialização manual com `systemctl start`.

O teste realizado confirmou a recuperação após um encerramento forçado. O processo Python, inicialmente identificado pelo PID 68993, foi encerrado com o sinal `SIGKILL`. Os registros do Journal mostraram que a falha ocorreu às 10:47:08. Às 10:47:13, cinco segundos depois, foram registrados o agendamento da tentativa de reinicialização e o início de uma nova instância do serviço. Após a recuperação, a aplicação voltou a executar com o PID 69040.

Outro benefício foi a configuração de inicialização automática por meio de `systemctl enable`. Dessa forma, o serviço fica associado ao `multi-user.target` e está configurado para iniciar automaticamente durante a inicialização do sistema, sem depender de uma execução manual.

Apesar dessas vantagens, a recuperação automática não garante que a aplicação esteja funcionando corretamente em todas as situações. O Systemd consegue identificar o encerramento do processo, mas não necessariamente problemas internos que mantenham o processo ativo, mesmo que a aplicação apresente erros ou deixe de responder. Para isso, seriam necessários mecanismos adicionais de monitoramento e verificação de saúde (health checks).

Também é importante destacar que o teste realizado simulou o encerramento forçado do processo, mas não uma falha real por falta de memória ou uma reinicialização completa da máquina. Portanto, o que foi efetivamente comprovado foi a recuperação automática após o encerramento por `SIGKILL`. A inicialização após um reboot e a recuperação diante de uma situação real de falta de memória não foram testadas.

De modo geral, a atividade demonstrou como o Systemd pode melhorar a disponibilidade de um serviço, centralizando seu gerenciamento, permitindo a inicialização automática e oferecendo mecanismos para recuperar processos encerrados.

O limite de inicializações do Systemd pode ser personalizado pelas diretivas `StartLimitIntervalSec` e `StartLimitBurst`, configuradas na seção `[Unit]`. Com `RestartSec=5`, uma sequência de falhas imediatas não atinge o limite padrão de cinco tentativas em dez segundos. Esse limite passa a ser mais relevante quando o intervalo entre as tentativas é reduzido, aumentando a frequência das reinicializações.

### 5. Conferência dos requisitos da atividade

| Requisito | Evidência |
|---|---|
| Criar `/etc/systemd/system/app-producao.service` | Arquivo criado e conteúdo conferido com `cat`. |
| Configurar `[Unit]` | `Description` e `After=network.target`. |
| Definir usuário e diretório | `User=evalle` e `WorkingDirectory`. |
| Configurar inicialização Python | `ExecStart` com `/usr/bin/python3`. |
| Configurar recuperação automática | `Restart=always` e `RestartSec=5`. |
| Configurar inicialização automática | `WantedBy=multi-user.target`. |
| Recarregar o Systemd | `daemon-reload` executado. |
| Iniciar e habilitar o serviço | `enable --now` executado. |
| Consultar status e logs | `systemctl status` e `journalctl` executados. |
| Testar recuperação automática | `SIGKILL` seguido de reinicialização confirmada nos logs. |

---

## Atividade 4: Experimentação com Inteligência Artificial

### Registro da consulta

- **Ferramenta utilizada:** Claude, modelo Haiku 5.5.
- **Data:** 14/11/2026.
- **Objetivo:** compreender as diferenças arquiteturais entre Cron e Systemd, com foco em tarefas em lote e serviços persistentes em segundo plano.

### Prompt utilizado

> No Linux Ubuntu, qual é a diferença arquitetural entre agendar scripts periódicos via Cron e gerenciar processos contínuos via Systemd? Quando uma tarefa de backend deve ser um Cron job e quando deve ser um Systemd Service?

### Análise: diferenças entre Cron e Systemd

Os jobs em lote (*batch*) são tarefas executadas para realizar uma operação específica, geralmente com início e fim definidos. Após concluir o processamento, o programa encerra sua execução, aguardando um novo agendamento para ser executado novamente.

Já os serviços persistentes em segundo plano são processos que permanecem ativos continuamente, aguardando requisições, executando operações ou disponibilizando funcionalidades para outros sistemas.

O Cron é um agendador de tarefas, utilizado principalmente para executar comandos em horários ou intervalos predefinidos. É indicado para tarefas em lote, como realização de backups, limpeza de arquivos temporários, manutenção de logs e geração de relatórios.

Uma característica do Cron é que ele não supervisiona continuamente o processo depois que a tarefa é iniciada. Se uma execução apresentar uma falha, o Cron não tenta reiniciá-la automaticamente. A próxima execução ocorrerá somente no horário programado. Também é necessário tomar cuidado com tarefas demoradas, pois uma nova execução poderá começar antes que a anterior termine. Para evitar essa sobreposição, podem ser utilizadas ferramentas como `flock`, que permitem controlar a execução simultânea de um mesmo processo.

O Systemd, por outro lado, é um gerenciador de serviços e processos do sistema operacional. É utilizado principalmente para aplicações que precisam permanecer em execução em segundo plano, como APIs, servidores web, aplicações de backend e trabalhadores de filas (*workers*). Além de iniciar os processos, o Systemd acompanha seu estado e permite configurar políticas de reinicialização, inicialização automática junto com o sistema, dependências, limites de recursos e centralização de registros no Journal.

Uma diferença importante é que o Systemd permite definir o comportamento esperado quando um processo é encerrado. Por meio de diretivas como `Restart=always` e `Restart=on-failure`, é possível determinar quando o serviço deverá ser iniciado novamente. Entretanto, é necessário configurar corretamente essas políticas e investigar a causa de falhas recorrentes para evitar ciclos de reinicialização.

### Comparação entre as ferramentas

| Característica | Cron | Systemd |
|---|---|---|
| Finalidade principal | Agendar tarefas | Gerenciar serviços e processos |
| Tipo de execução | Periódica ou programada | Contínua, em segundo plano |
| Duração | Normalmente limitada | Geralmente permanece ativo |
| Supervisão contínua | Não | Sim |
| Recuperação automática | Não reinicia automaticamente a tarefa após falha | Permite configurar políticas de reinicialização |
| Inicialização com o sistema | Suportada por `@reboot`, mas não é o foco | Suportada por meio de `enable` |
| Registro de execução | Envia a saída para e-mail local por padrão ou para arquivo quando há redirecionamento | Integração com o Journal |
| Exemplos | Backup, limpeza e relatórios | API, servidor web e worker |

### Quando utilizar cada ferramenta

A escolha depende principalmente da finalidade e do comportamento esperado da aplicação.

O Cron deve ser utilizado quando existe a necessidade de executar uma tarefa em horários determinados, sem que o processo precise permanecer ativo durante todo o tempo. Um exemplo é o script de manutenção de logs desenvolvido na atividade anterior, que realiza uma cópia do arquivo `access.log` diariamente à meia-noite.

Já o Systemd é mais adequado quando a aplicação precisa permanecer disponível continuamente, aguardando requisições ou realizando processamento em segundo plano. Na atividade anterior, foi criado um serviço para manter o servidor HTTP do Python em execução, com inicialização automática e recuperação após encerramento inesperado.

Também existe uma alternativa chamada *Systemd Timers*, que permite agendar tarefas utilizando a própria estrutura do Systemd. Essa opção pode substituir o Cron em determinados cenários, oferecendo integração com o gerenciamento de serviços, dependências e registros no Journal.

### Conclusão

A principal diferença entre as duas ferramentas está na finalidade de cada uma: o Cron responde à necessidade de definir quando uma tarefa deve ser executada, enquanto o Systemd permite gerenciar processos que precisam permanecer ativos e disponíveis.

As ferramentas não são concorrentes, mas complementares. Em um ambiente de backend, por exemplo, uma API pode ser gerenciada pelo Systemd enquanto tarefas periódicas de backup, limpeza e manutenção são executadas pelo Cron.

As atividades práticas demonstraram essa diferença: o Cron foi utilizado para programar a manutenção dos logs, enquanto o Systemd assumiu o gerenciamento de um processo contínuo, permitindo sua inicialização automática e recuperação após uma interrupção forçada.

**Observação:** no Cron, o redirecionamento `>> arquivo 2>&1` permite armazenar tanto a saída padrão quanto as mensagens de erro em um arquivo. Sem esse redirecionamento, o Cron normalmente tenta encaminhar a saída por e-mail ao usuário responsável pela tarefa, dependendo da configuração de e-mail local.
