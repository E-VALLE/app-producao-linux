# Prática 01: Navegação, Inspeção e Deploy Atômico

## Observações do terminal

- `backend/` não existia em `~/app-producao`, por isso `ls -l backend/` falhou no início.
- O link `current-api` já existia e apontava para `backend/api`. Como o caminho não existia, o link estava **quebrado**.
- Não há `.git/` em `~/app-producao`. Isso será verificado na Atividade 2, etapa 3.

## Atividade 1

### 1. Diferença entre `ls` e `ls -la`

O `ls` mostra só os nomes dos arquivos e pastas que não começam com ponto. O `ls -la` mostra mais detalhes (`-l`: permissões, dono, tamanho, data e, em links, o destino com `->`) e também os itens ocultos (`-a`, de *all*).

A flag `-a` mostrou os itens que começam com ponto, como `.` (a pasta atual) e `..` (a pasta de cima). Arquivos de configuração e do Git, como `.git`, `.env` e `.gitignore`, também ficam ocultos e só aparecem com `-a`.

### 2. O que o `ln -s` fez e por que ele é útil em deploy

O comando `ln -s backend/api current-api` cria um link simbólico: um atalho que aponta para outro caminho, sem copiar nada. Como o link `current-api` já existia, o comando não criou nada e mostrou a mensagem `ln: Already exists`.

No deploy, esse atalho é usado para trocar de versão sem parar o site. O servidor sempre aponta para o mesmo nome (`current`), e cada versão fica em uma pasta própria:

1. a nova versão é preparada em outra pasta, sem mexer na que está no ar;
2. o atalho `current` é trocado para apontar para a nova pasta. Essa troca é quase instantânea;
3. se algo der errado, o atalho volta para a versão anterior.

Como a troca é uma única operação, o site não fica com arquivos pela metade. Se a aplicação mantém código em memória, pode ser preciso reiniciar o serviço depois da troca.

### 3. Organização da estrutura do projeto

Para deixar a estrutura no nível exigido pelo roteiro, usei os comandos abaixo, em sequência:

1. `cd ~` e `mv -nv current-api app-producao/`
   Movi o link de volta para dentro de `~/app-producao`, onde ele deveria estar.
   Raciocínio: o link foi movido por engano para `~` em uma tentativa anterior. Voltar para a pasta do projeto é a correção.

2. `cd ~/app-producao` e `mv -nv app-producao/* .`
   Movi os itens da pasta interna para `~/app-producao`. O `-n` impede sobrescrever algo que já exista.
   Raciocínio: como a pasta de destino é a atual (`.`), os arquivos ficam no nível esperado pelo roteiro.

3. `rmdir app-producao`
   Removi a pasta interna, que já estava vazia. O `rmdir` só remove pastas vazias, então não há risco de apagar conteúdo.

4. `ls -la`, `ls -l current-api` e `ls -l backend/`
   Confirmei que `backend/` está em `~/app-producao` e que `backend/api` existe. Assim, o link `current-api` passou a apontar para um caminho válido.

**Erro registrado:** na primeira tentativa, rodei `mv -nv app-producao/* .` a partir de `~`, e não de `~/app-producao`. O resultado foi que o link `current-api` foi movido para `~`. O erro foi corrigido movendo o link de volta, e o projeto não foi afetado.

## Atividade 2

### Etapa 1: remover os arquivos `.tmp`

Comandos usados, a partir de `~/app-producao`:

1. `find -name '*.tmp' -type f`
   Mostrou três arquivos: `./app-producao/temp_build.tmp`, `./app-producao/backend/temp_debug.tmp` e `./app-producao/logs/cache_antigo.tmp`.
   Raciocínio: primeiro listo para ver o que será apagado. O `-type f` garante que só arquivos entrem na lista, e não pastas com nome parecido.

2. `rm ./temp_build.tmp backend/temp_debug.tmp logs/cache_antigo.tmp`
   Falhou com "Arquivo ou diretório inexistente". Nada foi apagado.
   Raciocínio: usei caminhos a partir da pasta errada. Como o `rm` só remove o que existe no caminho informado, o erro não causou perda. Mesmo assim, registro o erro.

3. `find . -name '*.tmp' -type f`
   Confirmou que os três arquivos continuavam lá, com os caminhos corretos.

4. `rm ./app-producao/temp_build.tmp ./app-producao/backend/temp_debug.tmp ./app-producao/logs/cache_antigo.tmp`
   Executou sem erro.
   Raciocínio: usei os caminhos exatos que o `find` mostrou, sem usar curingas. Assim, só os três arquivos foram apagados.

5. `find . -name '*.tmp' -type f`
   Saída vazia. Confirma que não restou nenhum `.tmp` no projeto.

### Etapa 2: `backend/.env`

Comandos usados:

1. `ls -l app-producao/config/env.template`
   Arquivo existe, 138 bytes.

2. `ls -l app-producao/backend/.env`
   Arquivo **já existe**, 164 bytes, criado antes da prática.

3. `cat app-producao/config/env.template` e `cat app-producao/backend/.env`
   Comparei o conteúdo. Os valores de senha não serão registrados aqui.

4. `diff app-producao/config/env.template app-producao/backend/.env`
   Mostrou três diferenças: a linha `DATABASE_URL` tem usuário e senha reais no `.env`, e o `.env` tem a linha `MAX_REQUEST_LIMIT=50`, que não existe no modelo.

**Decisão: manter o `backend/.env` atual, sem copiar o modelo.**

Raciocínio: o `.env` já tem configuração específica deste ambiente. Copiar o modelo com `cp` sobrescreveria essa configuração sem aviso, e a variável `MAX_REQUEST_LIMIT` seria perdida.

Observação de segurança: o arquivo está com permissão `-rw-r--r--`, que permite leitura para outros usuários do sistema. Para um arquivo com senha, o recomendado seria `chmod 600`. Não executei essa alteração, por não fazer parte do roteiro.

**Nota de caminho:** os comandos desta etapa foram rodados com `app-producao/` no início, porque ainda estavam dentro da pasta interna. Depois da reorganização, o caminho passa a ser `backend/.env` a partir de `~/app-producao`.

### Etapa 3: `.gitignore`

Comandos usados, a partir de `~/app-producao`:

1. `git status` e `git rev-parse --show-toplevel`
   Resultado: "not a git repository". Não havia repositório Git em nenhum nível acima do projeto.
   Raciocínio: antes de criar qualquer arquivo, precisava saber se já existia um repositório. Se o `.gitignore` fosse criado no lugar errado, o Git o usaria a partir de outra pasta.

2. `git init`
   Criou o repositório vazio em `/home/evalle/app-producao/.git/`.
   Raciocínio: o projeto precisa estar versionado na pasta onde estão os arquivos. Essa pasta é a raiz do projeto.

3. `git status`
   Mostrou os itens do projeto como não monitorados. Nenhum `.tmp` aparece na lista, o que confirma a etapa 1.
   Raciocínio: esta é a situação antes do `.gitignore`. Serve como referência para comparar depois.

4. `ls -la logs/`
   Encontrei `access.log`, `error.log` e `.gitkeep`.
   Raciocínio: a regra `logs/*.log` precisa corresponder a arquivos reais. O `.gitkeep` não deve ser ignorado, porque mantém a pasta `logs/` no repositório.

5. `nano .gitignore`, com as linhas `*.tmp`, `logs/*.log` e `backend/.env`.
   Raciocínio: cada linha corresponde a um item do enunciado. Escrevi os padrões de forma específica, para não ignorar arquivos que não deveriam ser ignorados.

6. `cat .gitignore`
   Confirmou o conteúdo das três linhas.

7. `git status`
   Passou a mostrar `.gitignore` como não monitorado. Os arquivos ignorados não aparecem na lista.

8. `git check-ignore -v backend/.env logs/access.log logs/error.log`
   Resultado:
   - `backend/.env` foi bloqueado pela linha 3 do `.gitignore`;
   - `logs/access.log` e `logs/error.log` foram bloqueados pela linha 2.

   Raciocínio: o `git check-ignore` confirma, linha a linha, que cada padrão funciona. O `.gitkeep` não aparece nessa lista, então continua sendo versionado.

**Observação:** como o repositório ainda não tem nenhum commit, nenhum arquivo sensível chegou ao histórico. Por isso não foi preciso usar `git rm --cached`. Se o `backend/.env` tivesse sido commitado antes, o `.gitignore` sozinho não bastaria, porque o Git continuaria rastreando o arquivo.

## Atividade 3: Filtros de tempo e tamanho no `find`

Fonte consultada: manual oficial do GNU findutils (`find`), em gnu.org.

### 1. Arquivos modificados nos últimos 7 dias em `logs/`

Comando:

```bash
find logs/ -type f -mtime -7
```

Explicação:
- `logs/` é o diretório onde a busca começa.
- `-type f` limita a lista a arquivos, sem incluir a própria pasta `logs/`.
- `-mtime -7` seleciona os arquivos cujo conteúdo foi alterado há menos de 7 dias de 24 horas.

Observação: hoje os arquivos de `logs/` têm data de 8 de outubro, ou seja, há cerca de 23 dias. Por isso o comando deve retornar uma lista vazia agora. Ele só listará arquivos se algo for gravado em `logs/` nos próximos 7 dias.

### 2. Diferença entre `-mtime -7` e `-mtime +7`

O manual diz que o número de dias é sempre arredondado para baixo, e que `-n` e `+n` são faixas.

- `-mtime -7` seleciona arquivos modificados há **menos de 7 dias**, ou seja, com idade igual a 0 a 6 dias completos.
- `-mtime +7` seleciona arquivos modificados há **mais de 7 dias**, ou seja, com idade de 8 dias completos em diante.

A diferença técnica é que `-7` é uma faixa de recência e `+7` é uma faixa de antiguidade. Um arquivo com exatamente 7 dias completos não entra em nenhum dos dois. Por isso, para limpar arquivos antigos, `+7` é a opção; para listar o que mudou recentemente, `-7`.

### 3. Filtro de tamanho para arquivos maiores que 10 MB

Comando:

```bash
find logs/ -type f -size +10M
```

Explicação:
- `-size +10M` seleciona arquivos com mais de 10 unidades de tamanho.
- O sufixo `M` significa **MiB** (mebibyte, 1 MiB = 1.048.576 bytes), não MB decimal.
- O manual diz que o tamanho é arredondado para cima até a unidade inteira. Por isso, um arquivo de 10,2 MiB conta como 11 MiB para essa comparação.

Se o critério for 10 MB decimal (10.000.000 bytes), o filtro exato em bytes é:

```bash
find logs/ -type f -size +10000000c
```

O sufixo `c` indica bytes, sem arredondamento de unidade.

