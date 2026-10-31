# Prática 02: Privilégios, edição em modo texto e análise forense de logs

## Atividade 1: Hardening de permissões

Entrei na pasta do projeto e rodei os comandos do roteiro:

- `chmod 755 config logs backend frontend`
- `chmod 644 backend/api/server.js config/env.template`
- `chmod 600 backend/.env`

Depois conferi com `ls -ld backend` e `ls -l backend/.env`. A pasta `backend/` passou de `drwxrwxr-x` para `drwxr-xr-x`, e o `.env` foi de `-rw-r--r--` para `-rw-------`.

### Análise

**Máscara 600.** Cada dígito é a soma de 4 (ler), 2 (escrever) e 1 (executar), e os dígitos seguem a ordem dono, grupo e outros. O `6` do dono é 4 + 2, então ele lê e escreve. Grupo e outros têm `0`, sem acesso nenhum.

**Por que o `.env` precisa de 600.** Ele guarda a senha do banco, e quem consegue ler o arquivo consegue acessar o banco. Com 600, só o dono tem acesso.

**Por que 777 é grave.** Com 777, qualquer usuário pode alterar e executar os arquivos. Se a aplicação tiver uma falha de upload, um invasor pode gravar um script na pasta e executá-lo. O correto é ajustar o dono e o grupo, dando ao servidor web só o acesso de que ele precisa.

### Pendências

- Verificar qual usuário executa a API: `ps -eo user,pid,cmd | grep -E "node|api" | grep -v grep`.
- Rodar `git status` para ver se o `chmod 644` mudou o bit de execução de algum arquivo.

## Atividade 2: Edição em modo texto

Abri o arquivo com o `nano`:

```bash
nano backend/.env
```

Troquei `PORT` para `8080` e acrescentei `MAX_REQUEST_LIMIT=100`. Para salvar, usei `Ctrl + O` e `Enter`. Para sair, `Ctrl + X`. Depois conferi com `cat backend/.env`, e as duas alterações estavam lá.

## Atividade 3: Mineração forense de logs

Antes de tudo, conferi o formato do log com `head -3 logs/access.log`. Cada linha tem o IP no campo 1, o caminho no campo 6 e o status no campo 8. Confirmei com `awk '{print $1, $6, $8}' logs/access.log | head -3`.

**1. Força bruta no login**

```bash
awk '$6 == "/api/v1/login" && $8 == 401 {print $1}' logs/access.log | sort | uniq -c | sort -rn | head -1
```

O `awk` mantém só as linhas com caminho `/api/v1/login` e status `401`, e imprime o IP. O `sort` agrupa os IPs iguais, e o `uniq -c` conta quantas vezes cada um aparece. O `sort -rn` ordena do maior para o menor, e o `head -1` mantém o primeiro.

Resultado: o IP `192.168.1.15` fez **10 tentativas** com erro 401.

**2. Erros 500**

```bash
awk '$8 == 500' logs/access.log | wc -l
```

O `awk` filtra as linhas com status `500` e o `wc -l` conta quantas sobraram.

Resultado: **4 requisições** receberam erro 500.

**3. Relatório de IPs com erros 4xx e 5xx**

```bash
awk '$8 ~ /^[45]/ {print $1}' logs/access.log | sort | uniq -c | sort -rn > logs/relatorio-incidentes.txt
```

O `awk` seleciona os status que começam com 4 ou 5. A lista é contada e ordenada como no item 1, e o `>` grava o resultado em `logs/relatorio-incidentes.txt`.

Como o relatório traz IPs de clientes, ele não deve ser publicado no repositório. Antes do próximo commit, é preciso adicionar `logs/*.txt` ao `.gitignore`, ou deixar esse arquivo fora do `git add`.

## Atividade 4: Edição em lote com `sed`

**1. Trocar a porta sem abrir o editor**

```bash
sed -i 's/PORT=8080/PORT=3000/' backend/.env
grep PORT backend/.env
```

O `sed -i` faz a alteração direto no arquivo. O `s/antigo/novo/` troca o texto encontrado, e só a primeira ocorrência em cada linha. O `grep` mostra a linha depois da troca, sem exibir o restante do `.env`, onde está a senha.

Antes de gravar, dá para testar sem o `-i`, que mostra o resultado na tela sem alterar o arquivo. Se quiser guardar uma cópia, use `sed -i.bak`, mas o `.bak` contém a senha e deve ser apagado depois.

**2. Por que `sed` é melhor que editor visual em CI/CD**

Um comando `sed` pode ser repetido exatamente igual em todo deploy, e ele não depende de uma pessoa abrindo o arquivo. Além disso, funciona em servidores e pipelines sem interface gráfica, como uma sessão SSH ou um job de CI. Como o comando fica escrito em um script, ele pode ser versionado e revisado, o que deixa a mudança rastreável. Um editor visual depende de cliques e digitação, que são difíceis de reproduzir e de auditar.

O cuidado é que o `sed -i` não pede confirmação. Por isso, em um pipeline, convém testar sem o `-i` e conferir o resultado depois.

## Teste prático de permissões

Fiz o teste com o usuário `evalle`, em uma pasta criada só para o experimento, fora do projeto (`~/teste-permissoes/uploads`).

Usei `mkdir -p` para criar a pasta e `ls -ld` para ver o estado inicial. Depois apliquei os modos 777, 770, 750 e 700 com `chmod`, conferindo com `ls -ld` após cada alteração. Por último, com a pasta em 700, criei um arquivo com `touch` e conferi com `ls -l`.

| Modo aplicado | Resultado (`ls -ld`) |
|---|---|
| Inicial | `drwxrwxr-x` (775), dono e grupo `evalle` |
| 777 | `drwxrwxrwx` |
| 770 | `drwxrwx---` |
| 750 | `drwxr-x---` |
| 700 | `drwx------` |

O que observei:

- O `chmod` alterou só as permissões. O dono e o grupo continuaram `evalle`.
- Com 700, consegui criar o `teste.txt` normalmente, porque sou o dono da pasta. O arquivo ficou com `664` (`rw-rw-r--`) por causa da máscara padrão (umask) do ambiente. O modo do arquivo criado depende do umask, e não do modo da pasta.
- Não testei com outros usuários. Por isso, as restrições para o grupo e para terceiros são uma interpretação dos modos octais: com 770, o dono e o grupo têm acesso completo; com 750, o grupo perde a escrita; com 700, só o dono tem acesso.
- Ao final, apaguei a pasta de teste com `rm -r`.

## Atividade 5: Experimentação com inteligência artificial

**Ferramenta utilizada:** Claude Code

**Prompt enviado:** "No Linux Ubuntu, por que a permissão `chmod 777` em diretórios de upload de um servidor web representa uma vulnerabilidade crítica de execução remota de código? Qual é a configuração correta de usuário, grupo e permissões octais para mitigar esse risco em aplicações web?"

**Síntese das conclusões:**

A utilização de chmod 777 em diretórios de upload representa um risco de segurança porque concede a qualquer usuário local permissão para ler, criar, modificar e excluir arquivos nesses diretórios. Como não existe restrição de escrita para os demais usuários, um usuário local também pode apagar arquivos pertencentes a outros processos, pois o diretório não possui o bit sticky.

Em aplicações web, esse acesso excessivo pode facilitar a exploração de vulnerabilidades de upload, principalmente quando o servidor permite executar arquivos enviados como scripts.

A proteção adequada depende da definição correta do proprietário e do grupo, da aplicação de permissões restritivas e do bloqueio da execução de scripts nos diretórios destinados aos arquivos enviados.

**Análise da recomendação:**

Considerando uma aplicação em que o servidor web é responsável por receber e armazenar os arquivos, adotaria www-data como proprietário e grupo do diretório de uploads. Essa configuração é adequada quando o próprio processo web precisa realizar as gravações.

Utilizaria as seguintes permissões:
- Diretórios: 750.
- Arquivos: 640.

Essa configuração permite que o proprietário tenha controle total, enquanto os integrantes do grupo podem ler arquivos e acessar diretórios, sem permissão de escrita. Para funcionar corretamente, o processo responsável pela leitura precisa pertencer ao grupo proprietário.

Caso existam processos diferentes que precisem compartilhar o acesso, podem ser utilizadas ACLs (Access Control Lists), que permitem conceder permissões específicas a determinados usuários ou grupos sem ampliar os privilégios dos demais.

Também armazenaria os uploads fora da raiz pública do servidor web, evitando que os arquivos sejam acessados diretamente por URLs.

Outro cuidado fundamental é impedir a execução de scripts nesse diretório.

No Apache com PHP-FPM, utilizaria uma regra no VirtualHost ou, quando permitido pela configuração do servidor, no .htaccess, para impedir que arquivos .php sejam encaminhados ao handler do PHP-FPM. Isso pode ser implementado com um bloco <FilesMatch> associado a uma regra de negação de acesso.

No Nginx com PHP-FPM, utilizaria um bloco location específico para o diretório de uploads, com uma regra que negue requisições a arquivos .php, impedindo seu encaminhamento ao interpretador. É importante observar a precedência das diretivas: um bloco de prefixo convencional pode perder para um bloco de expressão regular, como location ~ \.php$. Para evitar esse problema, utilizaria location ^~ no prefixo correspondente ao diretório de uploads, impedindo que as regras genéricas de processamento PHP tenham precedência sobre essa localização.

Já no Apache com mod_php, a diretiva php_flag engine off pode desabilitar a execução do PHP naquele contexto, desde que o módulo esteja carregado e a diretiva seja permitida. Essa diretiva não funciona com PHP-FPM.

Essas medidas são mais adequadas do que utilizar 775 ou 777, pois evitam conceder escrita a usuários que não precisam dela. A configuração correta das permissões também é mais confiável do que depender exclusivamente de scripts de criação de diretórios, que não garantem a proteção dos arquivos existentes ou posteriormente enviados.

**Conclusão:**

A proteção dos diretórios de upload deve fazer parte da arquitetura de segurança da aplicação, e não ser tratada apenas como um ajuste de permissões no Linux.

Adotaria como configuração padrão o proprietário e grupo www-data, permissões 750 para diretórios e 640 para arquivos, desde que os processos responsáveis pelo acesso estejam corretamente associados ao grupo. Também manteria os arquivos fora da raiz pública e bloquearia a execução de scripts enviados pelos usuários.

Essa combinação impede que outros usuários locais alterem ou excluam os arquivos e reduz a possibilidade de que um arquivo malicioso enviado à aplicação seja executado pelo servidor web.
