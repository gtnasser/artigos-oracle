### Modos de Log no Oracle Database: ARCHIVELOG e NOARCHIVELOG


Os Logs de Redo são arquivos que registram todas as operações DML (Data Manipulation Language) e DDL (Data Definition Language) realizadas, ou seja, operações com dados (insert, update, delete) e atualizações de objetos.

Cada vez que uma transação é executada, as alterações são primeiro registradas nos Logs de Redo antes de serem aplicadas aos arquivos de dados. Isso garante que, em caso de falha, as transações possam ser recuperadas e aplicadas novamente para manter a consistência  dos dados. Os Logs de Redo podem ser sobrescritos após serem arquivados ou podem ser gravados em arquivos físicos para serem utilizados posteriormente.

Existem dois modos de gerenciamento desses logs, denominados ARCHIVELOG e NOARCHIVELOG, que determinam como os Logs de Redo são armazenados e gerenciados. Esses modos influenciam diretamente a integridade dos dados e a capacidade de recuperação do banco de dados.

Compreender e utilizar corretamente esses modos é essencial para garantir a integridade, atualização e disponibilidade dos dados, especialmente em ambientes de produção onde a perda de informações pode ter consequências significativas, impactando diretamente a continuidade das operações.


#### Modo ARCHIVELOG

Permite a recuperação completa do banco de dados até o ponto de falha, pois todos os Logs de Redo gerados são gravados fisicamente em arquivos antes de serem sobrescritos. Isso significa que, mesmo após a recuperação de um backup completo feito antes do momento
 da ruptura, você ainda pode recuperar todas as transações realizadas após o backup até o momento desta ruptura.

**Uso Comum**: Ambientes onde a perda de dados impacta diretamente na continuidade da operação. Por exemplo:

  - **Ambientes de Produção**: Em sistemas de produção, onde a perda de dados pode ter um impacto significativo, o modo ARCHIVELOG é essencial para garantir que todas as transações sejam registradas e possam ser recuperadas em caso de falha.

  - **Backup Online**: Se você precisar realizar backups enquanto o banco de dados está em uso, o modo ARCHIVELOG permite que você faça isso sem interromper as operações.

  - **Recuperação de Desastres**: Em estratégias de recuperação de desastres, o modo ARCHIVELOG permite restaurar o banco de dados até o ponto de falha, minimizando a perda de dados.


#### Modo NOARCHIVELOG

Oferece uma operação mais simples e menos sobrecarga de armazenamento, pois os Logs de Redo são sobrescritos sem serem arquivados. Isso significa que você só pode recuperar o banco de dados até o último backup completo.

**Uso Comum**: Ambientes de desenvolvimento ou teste, onde a perda de dados é aceitável e a recuperação completa não é uma prioridade.


#### Resumo

| Característica       | ARCHIVELOG                          | NOARCHIVELOG                        |
|----------------------|-------------------------------------|-------------------------------------|
| **Recuperação de Dados** | Permite recuperação até o ponto de falha | Permite recuperação até o último backup |
| **Desempenho**       | Pode ter uma leve sobrecarga        | Pode ter desempenho ligeiramente melhor |
| **Complexidade**     | Requer mais gerenciamento e espaço  | Operação mais simples                |


### Comandos na prática


#### Verificação do Status Atual

verificar o status atual do modo de log:
```sql
SQL> SELECT log_mode FROM v$database;
```
listar detalhes do log de archive ativo:
```sql
SQL> archive log list;
```
listar detalhes da thread de archive ativa:
```sql
select destination, reopen_secs, register, status from v$archive_dest where status='VALID';

DESTINATION                              REOPEN_SECS REG STATUS
---------------------------------------- ----------- --- ---------
/u01/arch                                        300 YES VALID
```

#### Habilitar o Modo ARCHIVELOG

Parar o Banco de Dados:
```sql
SQL> shutdown immediate;
```
Iniciar em Modo MOUNT:
```sql
SQL> startup mount;
```
Habilitar ARCHIVELOG:
```sql
SQL> alter database archivelog;
```
Abrir o Banco de Dados para uso:
```sql
SQL> alter database open;
```


#### Desabilitar o Modo ARCHIVELOG

Parar o Banco de Dados:
```sql
SQL> shutdown immediate;
```
Iniciar em Modo MOUNT:
```sql
SQL> startup mount;
```
Desabilitar ARCHIVELOG:
```sql
SQL> alter database noarchivelog;
```
Abrir o Banco de Dados:
```sql
SQL> alter database open;
```


#### Alterar a Pasta de Destino

Se for necessário alterar a localização da pasta onde os arquivos serão gravados:
```sql
SQL> alter system set log_archive_dest_1='LOCATION=/home/oracle/arch' scope=both;
```


### Recuperar o backup e os dados arquivados

Após uma recuperação de backup no Oracle, deve-se realizar a aplicação dos Logs de Redo arquivados até o ponto de falha, antes que o banco de dados seja liberado para uso.

A recuperação de backup e a importação de dados são processos distintos no Oracle. De maneira geral, a importação dos dados do backup deve ser feita com o banco de dados no modo OPEN e a recuperação dos dados arquivados no REDO deve ser feita com o banco de dados no modo MOUNT.

-----

#### Passos gerais para importação de dados com o utilitário **imp**

Iniciar o Banco de Dados em Modo OPEN:
```sql
SQL> startup;
```
Importar os dados. Se necessário, criar o USER, SCHEMA e atribuir os GRANTS:
```sql
$ imp usuario/senha@banco file=backup.dmp full=y
```
Iniciar o Banco de Dados em Modo MOUNT:
```sql
SQL> startup mount;
```
Aplicar os Logs de Redo Arquivados:
```sql
SQL> recover database;
```
Abrir o Banco de Dados para uso:
```sql
SQL> alter database open resetlogs;
```

-----

#### Passos gerais para importação de dados com o utilitário RMAN

Conectar ao RMAN:
```sql
rman target /
```
Se for  necessário especificar um control file:
```sql
RMAN> restore controlfile from 'backup_location';
```
Iniciar o Banco de Dados em Modo MOUNT:
```sql
RMAN> alter database mount;
```
Restaurar o backup:
```sql
RMAN> restore database;
```
Restaurar os dados até o ponto de falha:
```sql
RMAN> recover database;
```
Abrir o banco de dados para uso:
```sql
RMAN> alter database open resetlogs;
```


-----


### Evoluções e Funcionalidades Adicionais

Embora os modos ARCHIVELOG e NOARCHIVELOG permaneçam essenciais, o Oracle introduziu várias funcionalidades adicionais para melhorar a gestão de logs e a recuperação de dados:

**Flashback Technology:**

Permite a recuperação de dados a um ponto específico no tempo sem a necessidade de restaurar a partir de um backup completo. Inclui recursos como Flashback Database, Flashback Table, e Flashback Query.

**Data Guard:**

Proporciona alta disponibilidade, proteção de dados e recuperação de desastres. Permite a criação de bancos de dados standby que podem ser sincronizados com o banco de dados primário.

**Recovery Manager (RMAN):**

Ferramenta robusta para backup e recuperação, que pode automatizar muitas das tarefas associadas à gestão de Logs de Redo e recuperação de dados.

**Automatic Storage Management (ASM):**

Simplifica a gestão de armazenamento e melhora o desempenho de I/O. Pode ser usado em conjunto com os modos ARCHIVELOG e NOARCHIVELOG para otimizar a gestão de logs.

**Conclusão**

Os modos ARCHIVELOG e NOARCHIVELOG continuam sendo a base para a gestão de Logs de Redo no Oracle. No entanto, as funcionalidades adicionais como Flashback, Data Guard, RMAN e ASM oferecem camadas extras de proteção e flexibilidade, permitindo uma recuperação de dados mais eficiente e robusta.

