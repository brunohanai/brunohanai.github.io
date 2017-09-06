---
layout: post
title: Tutorial básico de replicação Master-Slave no MySQL
category: MySQL
tags: [mysql]
---

## Introdução

A replicação do MySQL permite que os dados de um servidor principal (conhecido como **Master**) sejam copiados para outros
servidores (conhecidos como **Slave**).

Alguns motivos para usar a replicação são:

1. **Leitura:** Com os dados em outros servidores, liberamos as grandes consultas de dados (de relatórios, por exemplo)
do Master. Consultas (`select`) que exigem grande processamento podem causar travas ou lentidão no banco de dados, 
prejudicando as inserções (`insert`) e atualizações (`update`). Nesse cenário, `insert` e `update` são feitos no Master,
enquanto `select` de relatórios, dashboards e etc, são feitos no Slave.
2. **Backup:** Com as informações replicadas no Slave, o backup dos dados pode ser feito no Slave, sem travar o Master.
3. **Equipamento:** Um cliente deseja ter o banco de dados em um servidor próprio (que talvez seja mais poderoso). Ele, 
com a devida permissão e acesso, pode replicar os dados do servidor Master praticamente em tempo real. Com os dados no
próprio servidor pode realizar todos os tipos de análises que quiser sem prejudicar o sistema principal.

## Como funciona

Em uma replicação comum, temos dois servidores:

1. **Master:** o servidor principal
2. **Slave:** o servidor com a cópia do master

Podemos também ter cenários um pouco mais complexos, onde temos **Master-Slave(1)** e depois um **Slave(1)-Slave(2)**. Ou seja, 
um **Slave(2)** de um servidor que é **Slave(1)** de um **Master**.

De forma resumida, o processo de replicação funciona da seguinte forma:

1. Cada alteração executada no **Master** é escrita em um arquivo chamado "binary log".
2. Cada **Slave** configura as informações de conexão do **Master** (host, usuário, senha).
3. O **Slave** recebe uma cópia completa do "binary log".
4. Cada alteração que está no "binary log" escrita pelo **Master** é executada no **Slave**.

Para sincronizar esse processo, o **Master** armazena a posição atual do "binary log" e o **Slave** armazena a última 
posição que executou.

## Cenário 

Para esse tutorial, vamos considerar o seguinte cenário:

Temos 2 servidores, o **Master** e o **Slave**. Abaixo estão as informações de cada servidor:

* **Master:**
    - IP: 172.0.0.10
    - Usuário/Senha MySQL: user1/pass1 
    - Usuário/Senha para usar na replicação do MySQL: replicacao/replicacao
    - Nome do banco de dados MySQL: mydb
* **Slave:**
    - IP: 172.0.0.11
    - Usuário/Senha MySQL: user2/pass2
    
Considere essas informações como referência para entender melhor onde substituir pelas suas informações nas 
configurações e comandos abaixo.

## Passo a Passo

Abaixo vou explicar como faço a replicação. Considere que normalmente os servidores que utilizo estão com a versão 5.1 
do MySQL. 

Um guia que gosto de consultar quando preciso lembrar de algo é [esse aqui](http://download.nust.na/pub6/mysql/doc/refman/5.1/en/replication-howto.html).

### 1. Criar usuário MySQL para usar na replicação

O servidor **Master** precisa ter um usuário com a permissão "REPLICATION SLAVE" habilitada.

Os servidores **Slave** utilizarão esse usuário para se conectar ao **Master**.

No MySQL do **Master**:

<pre><code class="code sql">CREATE USER 'replicacao'@'%' IDENTIFIED BY 'replicacao';
GRANT REPLICATION SLAVE ON *.* TO 'replicacao'@'%';
FLUSH PRIVILEGES;
</code></pre>

### 2. Configurar arquivos my.cnf 

Vamos configurar os servidores **Master** e **Slave**.

**(Master) Arquivo "my.cnf":**

<pre><code class="code shell">[mysqld]
server-id               = 1
log-bin                 = /var/log/mysql/mysql-bin
max_binlog_size         = 1024M
expire_logs_days        = 10
</code></pre>

Aqui configuramos:

- `server-id`: Número único para identificar cada servidor que faz parte da replicação (ex: Master=1, Slave=2)
- `log-bin`: Diretório e prefixo do arquivo "binary log"
- `max_binlog_size`: Tamanho máximo do arquivo "binary log". Quando ultrapassar esse valor, o MySQL vai fechar o arquivo
atual e abrir um novo incrementando uma contagem (ex: "mysql-bin.000002")
- `expire_logs_days`: Número de dias que um arquivo "binary log" antigo pode permanecer no sistema (a limpeza acontece
no "start" do MySQL e quando o comando `FLUSH LOGS` é executado)

Vamos criar o diretório "/var/log/mysql" com a devida permissão de acesso:

<pre><code class="code shell">mkdir /var/log/mysql
chown mysql:mysql /var/log/mysql
</code></pre>

**(Slave) Arquivo "my.cnf":**

<pre><code class="code shell">[mysqld]
server-id               = 2
log-bin                 = /var/log/mysql-bin
max_binlog_size         = 1024M
expire_logs_days        = 10
master-host             = 172.0.0.10
master-port             = 3306
master-user             = replicacao
master-password         = replicacao
replicate-do-db         = mydb
replicate-ignore-db     = test
replicate-ignore-db     = mysql
log-slave-updates
</code></pre>

Aqui configuramos:

- `master-host`: O IP do servidor Master
- `master-port`: A porta do servidor Master
- `master-user`: Usuário criado no Master com a permissão de replicação
- `master-password`: Senha do usuário criado no Master com a permissão de replicação
- `replicate-do-db`: Para especificar bancos de dados que queremos fazer a replicação (podemos ter mais de uma linha 
com essa configuração para especificar outros bancos) 
- `replicate-ignore-db`: Para especificar bancos de dados que não queremos replicar
- `log-slave-updates`: Normalmente o Slave não escreve os updates que foram recebidos do Master em um "binary log". Essa
opção faz com que o Slave(1) crie e atualize o seu próprio "binary log", que depois pode ser utilizado para que um outro 
Slave(2) faça a replicação do Slave(1).

Obs: [Aqui tem um texto interessante](https://www.percona.com/blog/2009/05/14/why-mysqls-binlog-do-db-option-is-dangerous/)
sobre as opções "replicate-do-db" e "replicate-ignore-db". **Recomendo a leitura.**

Vamos criar o diretório "/var/log/mysql" com a devida permissão de acesso:

<pre><code class="code shell">mkdir /var/log/mysql
chown mysql:mysql /var/log/mysql
</code></pre>

Após a configuração, precisamos reiniciar tanto o Master como o Slave:

<pre><code class="code shell">service mysqld restart</code></pre>

Após isso o arquivo "binary log" já deve ter sido criado. Podemos verificar:

<pre><code class="code shell">ls -lh /var/log/mysql</code></pre>

Resposta do `ls`:

<pre><code class="code shell">-rw-rw----. 1 mysql mysql 106 Aug 8 16:52 mysql-bin.000001
-rw-rw----. 1 mysql mysql  32 Aug 8 16:52 mysql-bin.index
</code></pre>

### 3. Fazer o "dump" das informações do Master

No momento não temos nenhuma replicação ativa, ou seja, o Master tem todas as informações e o Slave nenhuma.

Para fazer o "start" da replicação, vamos colocar o Slave no mesmo momento do Master.

Fazemos isso exportando os dados atuais do Master e depois importando essas informações no Slave. A partir do momento
da importação, podemos ativar a replicação e as novas informações serão copiadas automaticamente.

Abaixo temos o passo a passo para esse processo. Tenha atenção no seguinte:

- Atenção em qual servidor cada comando deve ser executado
- Será preciso abrir dois terminais no Master pois um comando será executado em um terminal enquanto outro comando 
estará ativo em outro terminal 
- Verifique se a configuração do "iptables" dos servidores está correta!


**Master(terminal 1) (MySQL):**

<pre><code class="code sql">FLUSH TABLES WITH READ LOCK;</code></pre>

Esse comando irá bloquear todas as escritas executadas no Master (para que nenhuma escrita seja perdida enquanto estamos
fazendo a cópia do Master).

**Master(terminal 2):**

<pre><code class="code shell">mysqldump -u user1 -p mydb --master-data > dump.sql</code></pre>

Esse comando vai salvar os dados do servidor, do banco de dados "mydb", usando o usuário "user1", no arquivo "dump.sql".

A opção "--master-data" salva a posição atual do "binary log" do Master. Assim, quando fizermos a importação desse 
arquivo no Slave, ele automaticamente receberá a posição atual.

**Master(terminal 1) (MySQL):**

Quando o comando `mysqldump` terminar a exportação, podemos liberar as escritas no Master:

<pre><code class="code sql">UNLOCK TABLES;</code></pre>

**Master(terminal 1):**

Vamos conectar ao MySQL do Slave:

<pre><code class="code shell">mysql -h 172.0.0.11 -u user2 -p</code></pre>

Dentro do MySQL, vamos criar o banco de dados que queremos replicar e depois importar o arquivo de dump:

<pre><code class="code sql">CREATE DATABASE mydb;
USE mydb;
source dump.sql;
</code></pre>

Se o banco de dados do Slave não tiver acesso externo, será preciso enviar o arquivo para o servidor e executar os
comandos acima a partir de lá.

### 4. Iniciar e checar o status da replicação

E finalmente, vamos iniciar a replicação: 

<pre><code class="code sql">START SLAVE;</code></pre>

Para verificar se a replicação está ativa:

<pre><code class="code sql">SHOW SLAVE STATUS\G;</code></pre>

Abaixo as informações importantes do comando acima:

<pre><code class="code shell">*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.0.0.10
                  Master_User: replicacao
                  Master_Port: 3306
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 35295387
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: mydb
          Replicate_Ignore_DB: test,mysql
</code></pre>

- `Master_Host`, `Master_User` e `Master_Port`: Verificamos se a configuração de conexão está correta
- `Master_Log_File` e `Read_Master_Log_Pos`: Conferimos se as informações estão iguais ao do Master (executando `SHOW
MASTER STATUS` no Master)
- `Slave_IO_Running` e `Slave_SQL_Running` Status da replicação
- `Replicate_Do_DB` e `Replicate_Ignore_DB`: Verificamos se as configurações `replicate-do-db` e `replicate-ignore-db`
que fizemos no arquivo "my.cnf" estão corretas.

Uma última checagem é ver se alguma atualização que foi feita no Master após o processo do dump e o `START SLAVE` já foi
replicada no Slave.