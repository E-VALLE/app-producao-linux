# Prática 03: Processos e sinais

## Atividade 1: Execução em background e encerramento gracioso

### Sequência executada

```bash
cd app-producao
cd ~/app-producao
mkdir -p pids
python3 -m http.server 8080 --directory backend > logs/api.log 2>&1 &
echo $! > pids/api.pid
cat pids/api.pid
ps aux | grep "http.server 8080" | grep -v grep
kill -15 $(cat pids/api.pid)
ps -p $(cat pids/api.pid)
```

**Saída relevante:**

```text
[1] 45862
45862
evalle     45862  0.1  0.6  35412 22576 pts/0    S    09:20   0:00 python3 -m http.server 8080 --directory backend
    PID TTY          TIME CMD
[1]+  Terminado                  python3 -m http.server 8080 --directory backend > logs/api.log 2>&1
```

### Análise crítica

**1. O que os operadores `&` e `> logs/api.log 2>&1` fizeram na linha de execução da aplicação?**

O operador `&`, colocado ao final do comando, iniciou o servidor HTTP em segundo plano (*background*). Assim, o terminal ficou livre para receber novos comandos enquanto o servidor continuava em execução.

A expressão `> logs/api.log 2>&1` redirecionou as mensagens do processo para o arquivo `logs/api.log`. O símbolo `>` enviou a saída padrão (descritor 1, normalmente usada para mensagens comuns) para esse arquivo. Já `2>&1` fez a saída de erro (descritor 2) seguir o mesmo destino da saída padrão. Dessa maneira, tanto as mensagens normais quanto os erros ficaram registrados no mesmo arquivo, em vez de aparecerem no terminal.

**2. O que a variável `$!` capturou para ser salva em `pids/api.pid`?**

A variável especial `$!` contém o PID (*Process ID*, identificador do processo) do último processo iniciado em segundo plano pelo shell atual.

Nesse caso, depois que iniciei o servidor com `&`, `$!` recebeu o PID `45862`. O comando `echo $! > pids/api.pid` gravou esse número no arquivo. Em seguida, `cat pids/api.pid` mostrou o valor armazenado, e a listagem com `ps` identificou o processo em execução com o mesmo PID.

Guardar o PID facilita localizar e administrar o processo correto posteriormente, sem precisar procurar seu número manualmente entre todos os processos em execução.

**3. Qual é a diferença técnica entre o sinal SIGTERM (`kill -15`) e o sinal SIGKILL (`kill -9`)? Por que encerrar serviços de servidor web diretamente com `kill -9` é uma má prática?**

O `SIGTERM` é um pedido para que o processo encerre. Ele pode ser tratado pela aplicação, que pode executar procedimentos de finalização, como parar de aceitar novas requisições, concluir operações em andamento, fechar arquivos e liberar recursos. No exercício, `kill -15 $(cat pids/api.pid)` enviou esse sinal ao processo cujo PID estava registrado no arquivo.

O `SIGKILL`, por sua vez, força o encerramento do processo pelo sistema operacional. O processo não consegue capturar, ignorar ou tratar esse sinal e, portanto, não tem oportunidade de executar sua própria rotina de finalização.

Por isso, iniciar o encerramento de um servidor web diretamente com `kill -9` é uma prática ruim. Um encerramento forçado pode interromper requisições em andamento, deixar operações incompletas e impedir que a aplicação finalize tarefas ou libere recursos de forma controlada. A prática recomendada é tentar primeiro um encerramento gracioso com `SIGTERM` e recorrer ao `SIGKILL` somente se o processo não terminar dentro de um prazo razoável e após verificar que isso é realmente necessário.

Uma ressalva importante: enviar `SIGTERM` oferece à aplicação a oportunidade de tratar o encerramento, mas não garante, por si só, uma finalização graciosa. Isso depende de como o programa foi desenvolvido. Neste exercício, usei `python3 -m http.server` intencionalmente, seguindo o exemplo das videoaulas. Como a atividade aborda comandos básicos de administração de servidores Linux, o servidor HTTP simples atende ao objetivo de praticar execução em segundo plano, identificação de PID, redirecionamento de logs e encerramento por sinal. Em um cenário real, a aplicação seria uma API ou outro serviço próprio do ambiente.

---

## Atividade 2 — Desafio Prático: Diagnóstico de Hardware e Portas de Rede (Requisito 08)

### 1. Auditoria de memória RAM

**Comando utilizado:**

```bash
free -h
```

A opção `-h` (*human-readable*) apresenta os valores em unidades legíveis, como MiB e GiB.

**Saída obtida na VM:**

```text
               total       usada       livre    compart.  buff/cache  disponível
Mem.:          3,3Gi       1,5Gi       409Mi        24Mi       1,6Gi       1,8Gi
Swap:          3,8Gi       956Mi       2,9Gi
```

**Interpretação:** a VM possui aproximadamente `3,3 GiB` de RAM. No momento da consulta, `1,5 GiB` correspondiam à memória usada por processos e pelo kernel, sem contabilizar o cache; `409 MiB` estavam livres e `1,6 GiB` eram utilizados por buffers e cache.

Para avaliar a disponibilidade de memória, o campo `disponível` é mais útil do que `livre`, pois estima quanto o sistema ainda pode fornecer às aplicações considerando também a memória que pode ser recuperada de buffers e cache. Na consulta, esse valor era de aproximadamente `1,8 GiB`.

O Linux utiliza parte da RAM livre como cache para acelerar operações de leitura e escrita. Essa memória pode ser recuperada quando as aplicações precisam dela. A linha `Swap` indica que `956 MiB` do espaço de troca estavam em uso. Swap não é memória RAM: é espaço em disco utilizado como apoio quando necessário. Seu uso, isoladamente, não comprova falta de memória, mas merece acompanhamento se aumentar junto com a queda da memória disponível ou com sinais de lentidão.

### 2. Auditoria de armazenamento

**Comando utilizado:**

```bash
df -h /
```

**Saída obtida na VM:**

```text
Sist. Arq.      Tam. Usado Disp. Uso% Montado em
/dev/sda2        24G   12G   12G  50% /
```

**Interpretação:** a partição raiz `/dev/sda2`, onde está montado o sistema `/`, possui `24 GB` de espaço total. Metade do espaço (`12 GB`) estava ocupada e aproximadamente `12 GB` permaneciam disponíveis. O uso de `50%` indica que, no momento da consulta, a partição não apresentava ocupação elevada.

Também executei `df -h`, que lista os demais sistemas de arquivos montados. Essa consulta mostrou, entre outros itens, sistemas temporários (`tmpfs`) e a unidade `/dev/sr0` montada em `/mnt/cdrom`, com `100%` de uso. Essa unidade de CD-ROM não altera o resultado da auditoria da partição raiz; para o requisito solicitado, a consulta específica `df -h /` é a referência.

### 3. Auditoria de portas ocupadas

Na pasta do projeto, criei as pastas necessárias e iniciei o servidor HTTP de teste na porta `8080`, em segundo plano:

```bash
cd ~/app-producao
mkdir -p logs pids
python3 -m http.server 8080 --directory backend > logs/api.log 2>&1 &
echo $! > pids/api.pid
```

O terminal informou:

```text
[1] 46180
```

Assim, o PID retornado para o processo iniciado em segundo plano foi `46180` e ficou armazenado em `pids/api.pid`.

Para identificar o processo que estava escutando na porta TCP `8080`, executei:

```bash
sudo ss -ltnp 'sport = :8080'
```

**Saída obtida na VM:**

```text
State     Recv-Q    Send-Q        Local Address:Port         Peer Address:Port    Process
LISTEN    0         5                   0.0.0.0:8080              0.0.0.0:*        users:(("python3",pid=46180,fd=3))
```

**Interpretação:** o estado `LISTEN` confirma que havia um processo aguardando conexões TCP na porta `8080`. A saída identificou o processo `python3`, com PID `46180`, exatamente o mesmo PID informado ao iniciar o servidor e armazenado no arquivo `pids/api.pid`. O endereço `0.0.0.0:8080` indica que o servidor estava escutando na porta `8080` em todas as interfaces IPv4 da máquina.

Depois da confirmação, enviei o sinal de encerramento gracioso:

```bash
kill -TERM "$(cat pids/api.pid)"
```

O terminal apresentou:

```text
[1]+  Terminado                  python3 -m http.server 8080 --directory backend > logs/api.log 2>&1
```

Essa mensagem do shell confirmou que o trabalho em segundo plano terminou após o envio de `SIGTERM`.

### 4. Análise dos resultados e prontidão operacional

Considerando os resultados em conjunto, a VM apresenta condições adequadas para realizar o teste prático proposto: havia cerca de 1,8 GiB disponíveis no momento da medição para novas demandas, a partição raiz não estava com ocupação elevada e consegui iniciar o servidor de teste e confirmar que ele estava associado à porta TCP `8080`. Após a verificação, enviei `SIGTERM` e o shell informou o encerramento do processo.

**Conclusão:** considero a VM pronta para este teste introdutório de administração Linux. Essa avaliação não representa uma aprovação para uso em produção, pois ainda seria necessário medir o comportamento sob carga, acompanhar o crescimento dos logs e avaliar o tempo de resposta da aplicação.

---

## Atividade 3: Pesquisa Autônoma — Métricas de Memória no Top/Htop (Requisito 08)

### 1. Qual é a diferença conceitual entre as colunas VIRT (Virtual Image), RES (Resident Size) e SHR (Shared Memory)?

As três colunas apresentam perspectivas diferentes sobre a memória associada a um processo:

- **VIRT (Virtual Memory Size):** representa o tamanho total do espaço de memória virtual do processo. Inclui áreas de código, dados e bibliotecas compartilhadas, além de regiões mapeadas que ainda não foram utilizadas e páginas que foram transferidas para a área de swap. Portanto, esse valor não significa que toda essa memória esteja ocupando a RAM física naquele momento.

- **RES (Resident Memory Size):** representa a parte da memória do processo que está efetivamente residente na memória física, ou seja, que não está na swap naquele momento. Inclui páginas privadas do processo e também páginas compartilhadas que estão residentes na RAM.

- **SHR (Shared Memory Size):** representa uma parte da memória residente (`RES`) que pode ser compartilhada com outros processos. Pode incluir páginas de memória compartilhada e páginas associadas a arquivos, como bibliotecas e arquivos executáveis mapeados na memória.

No `htop`, a tela principal também apresenta essas métricas com os nomes `VIRT`, `RES` e `SHR`. Os identificadores `M_VIRT`, `M_RESIDENT` e `M_SHARE` aparecem na tela de configuração da ferramenta.

Além disso, `SHR` é um subconjunto de `RES`, e não um valor que deve ser somado a ele. Segundo o manual do `top`, `SHR` inclui páginas anônimas compartilhadas e páginas compartilhadas associadas a arquivos. Já `VIRT` inclui memória mapeada, mas ainda não utilizada, enquanto `RES` considera a memória física residente.

### 2. Ao monitorar um processo web em execução (Node.js ou Python), qual dessas três métricas reflete com maior precisão a quantidade real de memória física que o processo está ocupando na máquina?

Entre as três métricas, **`RES` é a mais adequada**, pois mostra a quantidade de memória física residente associada ao processo naquele momento. Diferentemente de `VIRT`, não inclui regiões virtuais mapeadas, mas ainda não utilizadas, ou páginas que já foram transferidas para a swap.

É importante observar que `RES` também inclui páginas compartilhadas. Por isso, seu valor não representa necessariamente a quantidade de RAM exclusiva daquele processo: uma mesma página compartilhada pode aparecer na memória residente de mais de um processo. A coluna `SHR` ajuda a identificar essa parcela compartilhável, mas não deve ser subtraída como se fosse uma medida exata de memória exclusiva em todos os casos.

**Conclusão:** para acompanhar o consumo de memória física de um processo web usando apenas essas três colunas, eu observaria principalmente `RES`. Se a intenção fosse estimar com mais precisão a parcela proporcional da memória compartilhada entre vários processos, seria interessante consultar também a métrica `PSS`, disponível em ferramentas e versões que oferecem essa informação.

### Fontes consultadas

- Manual do `top` — procps-ng, seção de descrição dos campos `VIRT`, `RES` e `SHR`: https://man7.org/linux/man-pages/man1/top.1.html. Acesso em: 7 nov. 2026.
- Manual do `htop` — descrição das métricas `M_VIRT`, `M_RESIDENT` e `M_SHARE`: https://man7.org/linux/man-pages/man1/htop.1.html. Acesso em: 7 nov. 2026.

---

## Atividade 4 — Experimentação com Inteligência Artificial

**Ferramenta utilizada:** Claude (Haiku 5.5)  
**Data da consulta:** 7 de novembro de 2026

**Prompt enviado:**

> No Linux Ubuntu, por que processos que são encerrados abruptamente com `kill -9` podem deixar portas TCP presas em estado TIME_WAIT ou conexões de banco de dados órfãs? Como uma aplicação Node.js ou Python deve tratar o sinal SIGTERM para realizar um encerramento gracioso (Graceful Shutdown)?

### Análise da explicação

A resposta da IA corrigiu uma premissa do prompt, em vez de aceitá-la integralmente. Ela explicou que o `kill -9` não causa necessariamente o estado `TIME_WAIT` e que esse estado não significa que o processo continua escutando na porta. `TIME_WAIT` é um estado temporário do protocolo TCP, administrado pelo kernel, que pode ocorrer durante o encerramento de uma conexão.

A resposta também poderia ser complementada com uma situação que realmente deixa uma porta em uso: quando um programa inicia outro processo, como um processo pai que inicia uma aplicação Node.js, e somente o processo pai é encerrado com `kill -9`, o processo filho pode continuar executando e mantendo a porta aberta. Isso acontece porque o sinal enviado ao pai não encerra automaticamente todos os processos filhos. Para evitar essa situação, é necessário que o gerenciamento dos processos também considere os filhos, por exemplo, por meio de um supervisor ou do encerramento adequado do grupo de processos.

Quanto às conexões de banco de dados, o `SIGKILL` impede que a aplicação execute sua rotina de encerramento e feche explicitamente o pool de conexões ou finalize operações pendentes. O sistema operacional fecha os descritores do processo, mas o banco de dados pode levar algum tempo para detectar a desconexão e liberar a sessão, especialmente quando há falhas de comunicação ou timeouts envolvidos.

### Resumo dos pontos aprendidos

- **`SIGKILL` e `SIGTERM`:** `SIGKILL` força o encerramento imediato, sem permitir que a aplicação execute sua rotina de limpeza. `SIGTERM` permite que ela receba o sinal e tente encerrar de forma controlada.
- **Estado `TIME_WAIT` e processos filhos:** `TIME_WAIT` é um estado TCP temporário e não significa que ainda exista um processo escutando na porta. Uma porta pode continuar em uso se um processo filho permanecer ativo após o encerramento do processo pai.
- **Conexões com o banco de dados:** sem um encerramento gracioso, a aplicação não consegue fechar explicitamente o pool nem concluir ou cancelar operações pendentes; o banco pode demorar para detectar e limpar a sessão.
- **Graceful Shutdown:** ao receber `SIGTERM`, a aplicação deve parar de aceitar novas requisições, aguardar as operações em andamento, fechar o servidor e as conexões com o banco e liberar os demais recursos, respeitando um prazo máximo para concluir o encerramento.


