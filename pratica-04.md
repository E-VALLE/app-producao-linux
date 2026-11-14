# Prática 04

## Atividade 1 – Execução HTTP e Diagnóstico com Shell Script

### Execução dos testes

Inicialmente, tentei iniciar manualmente o servidor HTTP na porta 8080:

```bash
cd ~/app-producao
python3 -m http.server 8080 --directory backend > logs/api.log 2>&1 &
```

Na primeira tentativa, o processo recebeu o PID 69671, mas encerrou sua execução. O comando `ss -tuln | grep 8080` não apresentou resultados, indicando que não havia nenhum processo escutando na porta naquele momento.

Ao executar:

```bash
curl -I http://localhost:8080/api/server.js
```

recebi a seguinte mensagem:

```text
curl: (7) Failed to connect to localhost port 8080 after 0 ms
```

Portanto, nessa tentativa, não foi possível estabelecer a conexão HTTP. O arquivo `logs/api.log` também estava vazio quando foi consultado.

Em outra tentativa de inicialização, foi identificado o erro:

```text
OSError: [Errno 98] Address already in use
```

Essa mensagem indica que a porta solicitada já estava sendo utilizada por outro processo. Entretanto, como o arquivo de log era sobrescrito a cada nova execução do comando, não foi possível confirmar que esse erro pertencia à mesma tentativa em que ocorreu a falha do `curl`.

#### Interrupção e reinicialização do serviço

Na primeira rodada de testes, também executei:

```bash
pkill -f "http.server 8080" 2>/dev/null || true
```

Esse comando encerrou o processo correspondente ao padrão informado, atingindo o servidor gerenciado pelo Systemd. Como o serviço estava configurado com `Restart=always`, o Systemd iniciou novamente o processo às 11:37.

Posteriormente, às 11:48, interrompi administrativamente o serviço para liberar a porta 8080:

```bash
sudo systemctl stop app-producao.service
```

#### Repetição do teste com a porta livre

Com a porta liberada, iniciei novamente o servidor HTTP em segundo plano:

```bash
python3 -m http.server 8080 --directory backend > logs/api.log 2>&1 &
```

Nessa execução, o processo recebeu o PID 69709.

Dessa vez, aguardei dois segundos antes de verificar a porta:

```bash
sleep 2
ss -tuln | grep 8080
```

O resultado foi:

```text
tcp LISTEN 0 5 0.0.0.0:8080 0.0.0.0:*
```

Isso confirmou que o servidor estava escutando na porta 8080. O endereço `0.0.0.0` indica que ele estava associado a todas as interfaces IPv4 da máquina.

Em seguida, executei:

```bash
curl -I http://localhost:8080/api/server.js
```

A resposta foi:

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.14.4
Date: Sat, 14 Nov 2026 14:51:35 GMT
Content-type: text/javascript
Content-Length: 989
Last-Modified: Thu, 08 Oct 2026 22:51:17 GMT
```

O código 200 OK confirmou que o arquivo solicitado foi localizado e disponibilizado corretamente pelo servidor HTTP.

### Configuração do ambiente

Para facilitar os próximos testes, adicionei ao arquivo `~/.bashrc` uma variável de ambiente e um alias:

```bash
export APP_API_URL="http://localhost:8080"
alias check-api='curl -s -I $APP_API_URL | head -n 1'
```

Depois, atualizei as configurações na sessão atual:

```bash
source ~/.bashrc
```

Executei o alias:

```bash
check-api
```

O resultado foi:

```text
HTTP/1.0 200 OK
```

Isso demonstrou que o alias estava funcionando corretamente. A resposta também indica que a variável `APP_API_URL` continha um endereço válido, pois, se estivesse vazia, o `curl` não teria recebido o endereço do servidor para realizar a requisição.

Ao finalizar os testes, encerrei o servidor iniciado manualmente:

```bash
kill $!
```

O comando encerrou o processo de teste de PID 69709, sem interromper diretamente o serviço gerenciado pelo Systemd.

### Inicialização e verificação posterior do serviço

Após finalizar o teste manual, iniciei novamente o serviço gerenciado pelo Systemd:

```bash
sudo systemctl start app-producao.service
```

Para verificar seu estado, executei:

```bash
systemctl status app-producao.service
```

O resultado apresentou:

```text
Active: active (running)
Main PID: 70026 (python3)
```

Também verifiquei a porta e realizei uma nova requisição HTTP:

```bash
sleep 3
sudo ss -tlnp 'sport = :8080'
curl -I http://localhost:8080/api/server.js
```

A verificação da porta apresentou:

```text
LISTEN 0 5 0.0.0.0:8080 0.0.0.0:* users:(("python3",pid=70026,fd=3))
```

E a requisição HTTP retornou:

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.14.4
Date: Sat, 14 Nov 2026 15:09:27 GMT
Content-type: text/javascript
Content-Length: 989
Last-Modified: Thu, 08 Oct 2026 22:51:17 GMT
```

O PID 70026 confirmou que o processo Python estava associado ao serviço `app-producao.service`, demonstrando que o servidor voltou a funcionar após sua inicialização pelo Systemd.

### Análise crítica

A atividade demonstrou a importância de verificar a disponibilidade da porta antes de iniciar um servidor HTTP. Na primeira rodada, a execução não foi bem-sucedida e o comando `curl` não conseguiu estabelecer conexão. Em outra tentativa, foi identificado o erro `Address already in use`, indicando conflito pelo uso da porta 8080. Após interromper o serviço gerenciado pelo Systemd e liberar a porta, foi possível iniciar o servidor manualmente e obter a resposta `HTTP/1.0 200 OK`.

Outro aprendizado importante ocorreu ao utilizar o comando `pkill -f "http.server 8080"`. Como o padrão informado correspondia tanto ao servidor iniciado manualmente quanto ao processo gerenciado pelo Systemd, o comando também encerrou o serviço. Isso demonstrou que comandos de encerramento baseados em padrões precisam ser utilizados com cuidado, principalmente quando existem processos com a mesma linha de execução. Para serviços gerenciados pelo Systemd, é mais adequado utilizar `systemctl stop` e `systemctl start`.

O teste também demonstrou como utilizar os comandos `ss` e `curl` para verificar, respectivamente, se uma porta está em estado de escuta e se um recurso está sendo disponibilizado corretamente por HTTP. A configuração da variável de ambiente `APP_API_URL` e do alias `check-api` no arquivo `~/.bashrc` facilitou a repetição dessa verificação.

Por fim, a inicialização posterior do serviço pelo Systemd e a identificação do PID `70026` confirmaram que o servidor voltou a funcionar sob gerenciamento do sistema. Dessa forma, a atividade permitiu compreender tanto o diagnóstico de serviços HTTP quanto a importância do gerenciamento adequado dos processos no Linux.

## Atividade 2 — Desafio Prático: Governança Criptográfica SSH

**Objetivo**

Preparar o ambiente Linux para autenticação SSH por meio de chaves assimétricas, utilizando o algoritmo Ed25519 e aplicando permissões restritivas aos arquivos de credenciais.

### 1. Geração das chaves

Inicialmente, foi criado o diretório `.ssh` e aplicada a permissão `700`:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Em seguida, foi gerado o par de chaves:

```bash
ssh-keygen -t ed25519 -C "deploy@app-producao.local" -f ~/.ssh/id_ed25519_app -N ""
```

O parâmetro `-t ed25519` seleciona o algoritmo Ed25519; `-C` adiciona um comentário identificador; `-f` define o caminho dos arquivos; e `-N ""` configura uma passphrase vazia, permitindo o uso não interativo em automações.

A geração produziu dois arquivos: `id_ed25519_app`, correspondente à chave privada, e `id_ed25519_app.pub`, correspondente à chave pública.

A impressão digital foi conferida com:

```bash
ssh-keygen -lf ~/.ssh/id_ed25519_app.pub
```

### 2. Hardening de permissões e fundamentação de segurança

Foram aplicadas as seguintes permissões:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519_app
chmod 600 ~/.ssh/authorized_keys
```

A permissão `700` no diretório `.ssh` concede leitura, escrita e execução somente ao proprietário. A permissão de execução é necessária para acessar e percorrer o diretório.

A permissão `600` na chave privada permite leitura e escrita somente pelo proprietário, impedindo o acesso direto de outros usuários locais ao conteúdo da credencial.

Essa configuração está relacionada ao mecanismo de segurança do OpenSSH. No lado do cliente, o programa `ssh` verifica as permissões da chave privada e pode ignorá-la quando identifica que está acessível a outros usuários, apresentando o aviso `WARNING: UNPROTECTED PRIVATE KEY FILE!`.

No lado do servidor, o processo `sshd` utiliza a opção `StrictModes`, habilitada por padrão, para verificar a propriedade e as permissões dos diretórios e arquivos envolvidos na autenticação. Se identificar condições inseguras, como permissões de escrita concedidas a usuários não autorizados, poderá rejeitar a autenticação por chave pública.

O arquivo `authorized_keys` recebeu permissão `600`, restringindo leitura e escrita ao proprietário. Embora uma permissão como `644` também possa ser aceita pelo `sshd`, por não conceder escrita aos demais usuários, foi adotada a configuração mais restritiva para reduzir a exposição das informações nele armazenadas.

O arquivo `id_ed25519_app.pub` permaneceu com permissão `644`, pois contém a chave pública, que não precisa ser mantida em segredo.

### Considerações sobre a passphrase

A chave privada foi gerada com o parâmetro `-N ""`, que define uma passphrase vazia. Essa escolha permite que processos automatizados de CI/CD utilizem a credencial sem depender de uma senha digitada interativamente.

Entretanto, a ausência de passphrase significa que a proteção adicional da chave por senha não existe. Nesse caso, a segurança depende principalmente das permissões restritivas do arquivo e do controle de acesso ao ambiente onde ele está armazenado. Se um usuário ou processo não autorizado conseguir obter a chave privada, poderá tentar utilizá-la para autenticação.

Em um ambiente de produção, esse risco deve ser reduzido por meio de uma chave exclusiva para implantação, armazenamento em um gerenciador de segredos do CI/CD, restrição dos privilégios da conta de deploy e procedimentos de rotação e revogação das credenciais.

### 3. Autorização de acesso

A chave pública foi cadastrada no arquivo `authorized_keys` por meio do comando:

```bash
cat ~/.ssh/id_ed25519_app.pub >> ~/.ssh/authorized_keys
```

O operador `>>` cria o arquivo caso ele ainda não exista ou acrescenta a chave ao conteúdo existente, preservando outras entradas. Como não foi realizada uma verificação prévia da existência de `authorized_keys`, não é possível afirmar qual dessas situações ocorreu durante a execução.

### 4. Auditoria da estrutura

A auditoria foi realizada com:

```bash
ls -la ~/.ssh
```

O resultado confirmou as permissões esperadas:

| Arquivo ou diretório | Permissão |
|---|---:|
| `.ssh` | `700` |
| `authorized_keys` | `600` |
| `id_ed25519_app` | `600` |
| `id_ed25519_app.pub` | `644` |

### 5. Conclusão

A atividade demonstrou a geração de um par de chaves Ed25519, o cadastramento da chave pública para autorização e a aplicação de permissões restritivas aos arquivos envolvidos na autenticação SSH.

Também permitiu compreender a diferença entre as verificações realizadas pelo cliente `ssh`, que protege o uso da chave privada, e pelo servidor `sshd`, que utiliza o mecanismo `StrictModes` para verificar a segurança dos arquivos e diretórios relacionados à autenticação.

A configuração atende aos requisitos do desafio e prepara o ambiente para autenticação automatizada. Entretanto, não foi realizado um teste de conexão SSH a partir de outra máquina. Portanto, a atividade comprova a preparação dos arquivos e das permissões, mas não a efetivação de uma autenticação remota.

## Atividade 3 — Pesquisa: Fortalecimento da Segurança do Serviço SSH

### 1. Objetivo

Pesquisar as principais diretivas de segurança do serviço OpenSSH, configuradas no arquivo `/etc/ssh/sshd_config`, e compreender a importância de proteger servidores Linux contra tentativas de acesso não autorizado.

### 2. Diretivas fundamentais de segurança

O arquivo `/etc/ssh/sshd_config` contém as configurações do servidor OpenSSH (`sshd`), responsável por permitir conexões remotas seguras. Entre as diretivas importantes para proteger servidores expostos à internet, destacam-se:

**a) PasswordAuthentication no**

Desabilita a autenticação tradicional por senha. Essa configuração impede que usuários utilizem senhas como método de autenticação por essa modalidade, reduzindo a exposição a tentativas automatizadas de descoberta de credenciais.

Na documentação do OpenBSD, o valor padrão dessa diretiva é `yes`. Portanto, configurá-la como `no` representa uma alteração de segurança em relação ao comportamento padrão (OPENBSD, s.d.).

**b) PermitRootLogin no**

Impede que o usuário root realize login diretamente pelo SSH. Essa medida reduz a exposição da conta administrativa principal. A recomendação é utilizar uma conta individual, com permissões controladas, e recorrer ao comando `sudo` quando forem necessárias operações administrativas.

O valor padrão informado pela documentação do OpenBSD é `prohibit-password`, que já impede a autenticação do usuário root por senha e por autenticação interativa. Entretanto, permite o acesso direto por chave pública. Ao configurar `PermitRootLogin no`, o servidor bloqueia completamente o login direto dessa conta, inclusive por chave pública (OPENBSD, s.d.).

**c) PubkeyAuthentication yes**

Habilita a autenticação por chave pública. Nesse método, o usuário utiliza um par de chaves criptográficas: a chave privada permanece protegida no computador do usuário, enquanto a chave pública é cadastrada no servidor, normalmente no arquivo `~/.ssh/authorized_keys`.

A autenticação ocorre por meio da comprovação criptográfica de posse da chave privada correspondente à chave pública cadastrada. Esse método é especialmente adequado para acessos administrativos e automatizados, desde que as chaves sejam protegidas e gerenciadas corretamente (YLÖNEN et al., 2015).

Segundo a documentação do OpenBSD, o valor padrão de `PubkeyAuthentication` já é `yes`. Assim, mantê-la habilitada permite utilizar esse método de autenticação, enquanto as outras duas diretivas restringem os acessos por senha e pela conta root (OPENBSD, s.d.).

As três diretivas podem ser configuradas no arquivo `/etc/ssh/sshd_config` da seguinte maneira:

```text
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

Essas configurações representam uma política de endurecimento de segurança, com restrições mais rigorosas para autenticação e acesso administrativo.

### 3. Riscos da autenticação por senha na porta 22

Manter a autenticação por login e senha habilitada em um servidor SSH exposto à internet aumenta a superfície de ataque, pois permite que usuários externos tentem descobrir credenciais válidas.

Entre as ameaças mais comuns estão:

- Ataques de força bruta: consistem em realizar diversas tentativas de autenticação, utilizando diferentes combinações de senhas até encontrar uma credencial válida.
- Ataques de dicionário: utilizam listas de senhas comuns, previsíveis ou obtidas de vazamentos anteriores para tentar acessar contas.
- Tentativas automatizadas: ferramentas e scripts podem executar repetidas tentativas de acesso, inclusive distribuídas entre diferentes endereços IP, dificultando a identificação e o bloqueio dos responsáveis.

O MITRE ATT&CK classifica a tentativa sistemática de descobrir senhas como Password Guessing (T1110.001). A documentação também identifica o SSH, que utiliza tradicionalmente a porta TCP 22, como um dos serviços frequentemente visados nesse tipo de ataque (MITRE, s.d.).

Caso uma senha seja descoberta, o invasor poderá obter acesso à conta comprometida e tentar explorar suas permissões para executar comandos, acessar arquivos ou avançar no comprometimento do servidor.

É importante destacar que o problema não está na criptografia do protocolo SSH, mas na possibilidade de explorar credenciais fracas, reutilizadas ou vulneráveis a tentativas repetidas de autenticação.

### 4. Considerações finais

A pesquisa demonstrou que a segurança do serviço SSH depende de configurações adequadas e de boas práticas no gerenciamento dos acessos.

Desabilitar a autenticação por senha, impedir o login direto do usuário root e habilitar a autenticação por chave pública são medidas importantes para reduzir os riscos de acesso não autorizado.

Além das configurações apresentadas, é necessário proteger as chaves privadas, controlar quem possui acesso ao servidor, manter o OpenSSH atualizado e monitorar os registros de autenticação.

A utilização dessas medidas contribui para tornar o acesso remoto mais seguro, especialmente em servidores expostos diretamente à internet.

### 5. Referências bibliográficas

OPENBSD. sshd_config(5) — OpenSSH daemon configuration file. OpenBSD Manual Pages. Disponível em: https://man.openbsd.org/sshd_config. Acesso em: 14 nov. 2026.

MITRE. Brute Force: Password Guessing (T1110.001). MITRE ATT&CK Enterprise. Disponível em: https://attack.mitre.org/techniques/T1110/001/. Acesso em: 14 nov. 2026.

YLÖNEN, Tatu; TURNER, Paul; SCARFONE, Karen; SOUPPAYA, Murugiah. Security of interactive and automated access management using Secure Shell (SSH). Gaithersburg: National Institute of Standards and Technology, 2015. (NIST Interagency/Internal Report, 7966). DOI: https://doi.org/10.6028/NIST.IR.7966. Acesso em: 14 nov. 2026.

## Atividade 4 — Experimentação com Inteligência Artificial

### 1. Objetivo

Utilizar uma ferramenta de Inteligência Artificial para compreender por que o cliente OpenSSH não permite o uso de uma chave privada quando suas permissões de arquivo são consideradas inseguras, relacionando esse comportamento ao princípio do menor privilégio.

### 2. Ferramenta de Inteligência Artificial utilizada

- Ferramenta: Claude
- Modelo: Claude Haiku 5.5 (`claude-haiku-5-5`)

Foi utilizado o seguinte prompt:

> No Linux, por que o cliente OpenSSH se recusa terminantemente a utilizar uma chave privada caso ela esteja com permissão 644 ou 777, exibindo o erro 'Permissions are too open'? Explique a fundamentação técnica de segurança defensiva por trás dessa decisão de projeto.

### 3. Análise da resposta da Inteligência Artificial

A resposta do Claude explica que uma chave privada SSH representa uma credencial de autenticação. Portanto, caso outros usuários do sistema consigam ler esse arquivo, poderão obter uma cópia da chave e utilizá-la para tentar acessar servidores que reconheçam a chave pública correspondente.

O cliente OpenSSH possui uma verificação de segurança para evitar essa situação. No código-fonte oficial, essa verificação é realizada pela função `sshkey_perm_ok`, localizada no arquivo `authfile.c`.

A função verifica se o arquivo pertence ao usuário que está executando o cliente SSH (`st_uid == getuid()`) e utiliza a máscara `077` para identificar se existem permissões concedidas ao grupo ou a outros usuários. Caso encontre alguma dessas permissões, considera o arquivo inseguro e apresenta mensagens como:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0644 for '/caminho/da/chave' are too open.
This private key will be ignored.
```

Essa regra não se limita às permissões 644 e 777. Ela também se aplica a outras combinações que concedam qualquer acesso ao grupo ou a outros usuários, como 640 e 620 (OPENSSH, s.d.).

É importante esclarecer que o cliente não necessariamente encerra toda a conexão quando encontra uma chave com permissões inadequadas. Ele ignora aquela chave e pode continuar tentando outros métodos de autenticação. Caso nenhum método alternativo funcione, a autenticação não será concluída.

### 4. Relação com o princípio do menor privilégio

O princípio do menor privilégio estabelece que cada usuário ou processo deve possuir somente as permissões necessárias para desempenhar suas funções.

No caso das chaves privadas SSH, isso significa permitir que somente o proprietário tenha acesso ao arquivo, impedindo que outros usuários do sistema possam ler ou modificar seu conteúdo.

Uma configuração bastante utilizada é a permissão `600`, aplicada pelo comando:

```bash
chmod 600 ~/.ssh/id_ed25519_app
```

Com essa configuração, o proprietário possui permissão de leitura e escrita, enquanto o grupo e os demais usuários não possuem acesso ao arquivo.

Essa restrição reduz o risco de exposição da credencial e demonstra como o OpenSSH aplica medidas preventivas de segurança, em vez de simplesmente emitir um aviso e continuar utilizando um arquivo potencialmente comprometido.

A verificação possui um escopo específico: a função `sshkey_perm_ok` aplica essa regra quando o arquivo pertence ao usuário que executa o cliente SSH. Portanto, não se deve interpretar essa proteção como uma barreira contra o usuário `root`, que possui privilégios administrativos e pode acessar arquivos de outros usuários.

### 5. Considerações finais

A análise permitiu compreender que as permissões de arquivos são fundamentais para proteger credenciais de autenticação no Linux.

A exigência do OpenSSH evita que uma chave privada seja utilizada quando suas permissões permitem acesso por outros usuários, reduzindo o risco de exposição e uso indevido da credencial.

Essa medida está diretamente relacionada ao princípio do menor privilégio, pois restringe o acesso à chave exclusivamente ao usuário que precisa utilizá-la. Dessa forma, a segurança não depende apenas da criptografia da chave, mas também da proteção adequada do arquivo em que ela está armazenada.

**Observação sobre o experimento:** a análise foi realizada com base na resposta da Inteligência Artificial e na consulta ao código-fonte oficial do OpenSSH. Não reproduzi o erro na máquina virtual; portanto, as mensagens apresentadas são exemplos documentados, e não resultados de um teste executado.

### 6. Referência bibliográfica

OPENSSH. `authfile.c` — OpenSSH Portable source code. GitHub, repositório oficial. Disponível em: https://github.com/openssh/openssh-portable/blob/master/authfile.c. Acesso em: 14 nov. 2026.
