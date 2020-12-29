# NewSQL - Tolerância à falha e escalabilidade com Cockroachdb e MemSQL

#### Autores
- Bernardo Pinheiro Camargo [@bernacamargo](https://github.com/bernacamargo)
- Renata Praisler [@RenataPraisler](https://github.com/RenataPraisler)

# 

## Objetivo
No contexto de bancos de dados relacionais e distribuídos (NewSQL), temos como objetivo deste projeto planejar e elaborar um tutorial intuitivo que permita a qualquer pessoa interessada testar e validar as características relacionadas a tolerância às falhas e escalabilidade na estrutura de NewSQL.

## Introdução

O NewSQL surgiu como uma nova proposta, pois com o uso do NOSQL acabou apresentando alguns problemas como por exemplo: a falta, do uso de transações, das consultas SQL e a estrutura complexa por não ter uma modelagem estruturada. Ele veio com o objetivo de ter os os pontos positivos dos do modelo relacional para as arquiteturas distribuídas e aumentar o desempenhos das queries de SQL, não tendo a necessidade de servidores mais potentes para melhor execução, e utilizando a escalabilidade horizontal e mantendo as propriedades ACID(Atomicidade, Consistência, Isolamento e Durabilidade).


## Tecnologias Habilitadores

- Kubernetes;
- Docker;
- Google Kubernetes Engine (GKE);
- Cockroachdb;
- MemSQL;

## 

## Cluster Kubernetes

Para podermos simular um ambiente isolado e que garanta as características de sistemas distribuídos utilizaremos um cluster local orquestrado pelo Kubernetes, o qual é responsável por gerenciar instâncias de máquinas virtuais para execução de aplicativos em containers. 

Primeiramente precisamos criar nosso cluster e utilizaremos o GKE para isto:

- Acesse a [Google Cloud Console](https://console.cloud.google.com)
- Navegue até o `Kubernetes Engine` e clique em `Clusters`;
- Clique em `Criar cluster` no centro da janela;
- Defina o nome do cluster e clique em `Criar`.

Feito isso, um cluster com três nós será criado e inicializado. Em alguns momentos você já poderá acessá-lo para seguirmos com as configurações.


> Para ambos os softwares Cockroachdb e MemSQL utilizaremos o mesmo processo para inicialização do cluster kubernetes, porém em clusters diferentes.

## Cockroachdb

Antes de iniciar o teste para identificar como é realizado a tolerância a falhas e a escalabilidade, temos que configurar o Cockroachdb no nosso cluster, para nos auxiliar utilizamos as documentações do Cockroachdb e kubernetes, e citaremos abaixo os comandos que devem ser realizados.

Para configurar a aplicação do cockroachdb dentro do cluster podemos fazer de algumas formas:
- [Usando o Operator](https://kubernetes.io/pt/docs/concepts/extend-kubernetes/operator/)
- [Usando o Helm](https://helm.sh/)
- Usando arquivos de configurações sem ferramentas automatizadoras.

Neste exemplo utilizaremos o Operator fornecido pelo Cockroachdb, pois ele automatiza a configuração da aplicação e assim não teremos que entrar a fundo em alguns detalhes mais técnicos do Kubernetes.

>Nota: É importante notar que temos um cluster kubernetes, composto de três instâncias de máquina virtual (1 master e 2 workers), onde as pods são alocadas e cada pod representa um nó do CockroachDB que está executando. Dessa forma quando falamos sobre os nós do cockroachdb estamos nos referindo as pods e quando falamos dos nós do cluster estamos falando das instâncias de máquina virtual do Kubernetes.

### 1. Instalar o Operator no cluster.

1.1. Criar o CustomResourceDefinition (CRD) para o Operator

```shell
$ kubectl apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/master/config/crd/bases/crdb.cockroachlabs.com_crdbclusters.yaml
```

O retorno esperado é:

    customresourcedefinition.apiextensions.k8s.io/crdbclusters.crdb.cockroachlabs.com created

> Nota: É interessante notar que o operator irá ser executado como uma pod do cluster.

1.2. Criar o Controller do Operator

```shell
$ kubectl apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/master/manifests/operator.yaml
```
    
O retorno esperado é:

    clusterrole.rbac.authorization.k8s.io/cockroach-operator-role created
    serviceaccount/cockroach-operator-default created
    clusterrolebinding.rbac.authorization.k8s.io/cockroach-operator-default created
    deployment.apps/cockroach-operator created

1.3. Validar se o Operator está rodando

```shell
$ kubectl get pods
``` 
Caso tenha funcionado, você deverá ter como retorno a seguinte mensagem:

    NAME                                 READY   STATUS    RESTARTS   AGE
    cockroach-operator-6867445556-9x6zp   1/1    Running      0      2m51s

> Nota: Caso o status da pod estiver como "ContainerCreating" é só aguardar alguns instantes que o kubernetes esta iniciando o container e logo deverá aparecer como "Running".

### 2. Configuração da aplicação do cockroachdb.

2.1. Realize o download e a edição do arquivo `example.yaml`, este é responsável por realizar a configuração básica de uma aplicação do cockroachdb através do Operator.
  
```shell
$ curl -O https://raw.githubusercontent.com/cockroachdb/cockroach-operator/master/examples/example.yaml
```

Utilize o `vim`(ou qualquer editor de texto de sua preferência) para abrir o arquivo baixado no passo anterior.

```shell
$ vim example.yaml
```
  
2.2. Com o arquivo aberto no editor de texto, vamos configurar a quantidade de CPU e memoria para cada pod do cluster. Basta procurar no arquivo baixado pelo código abaixo, descomentar as linhas e alterar os valores de `cpu` e `memory` para os que desejar.

```yaml
resources:
    requests:
        cpu: "2"
        memory: "8Gi"
        
    limits:
        cpu: "2"
        memory: "8Gi"
```

> Nota: Caso não defina nenhum valor inicial a aplicação extendera seus limites de uso de cpu/memoria até o limite do nó do cluster. 

> Nota: Essa etapa é opicional e contudo é recomendada em ambientes de produção, visto que limitar o uso de recurso na aplicação pode evitar um desperdício de recurso.
        
2.3. Modifique a quantidade de armazenamento cada pod terá, altere para quantos GigaBytes você desejar.
```yaml
resources:
    requests:
        storage: "1Gi"
```

> Nota: caso você queira outra configuração para outro teste ou projeto, se atente de verificar as [configurações recomendadas para a execução da aplicação cockroachdb](https://www.cockroachlabs.com/docs/v20.2/recommended-production-settings#hardware).

2.4. Aplique as configurações feitas no arquivo `example.yaml`.

```shell
$ kubectl apply -f example.yaml
```

O retorno esperado é:

    crdbcluster.crdb.cockroachlabs.com/cockroachdb created    

> Nota: Este arquivo irá solicitar para o Operador que crie uma aplicação StatefulSet com três pods que funcionarão como um cluster cockroachdb.

2.5. Verifique se as pods subiram.

```shell
$ kubectl get pods
```

O retorno esperado é:

    NAME                                  READY   STATUS    RESTARTS   AGE
    cockroach-operator-6867445556-9x6zp   1/1     Running   0          43m
    cockroachdb-0                         1/1     Running   0          2m29s
    cockroachdb-1                         1/1     Running   0          104s
    cockroachdb-2                         1/1     Running   0          67sa
    

### 3. Executando comandos SQL na pod.

Feito isso já temos nosso cluster e nossa aplicação configurados e executando. Agora chegou o momento de realizarmos nossos testes para averiguar a tolerância a falhas e a escalabilidade do CockroachDB.

#### 3.1. Acesse o bash de uma das pods que estão rodando a aplicação

```shell
$ kubectl exec -it cockroachdb-2 bash
```

> Nota: Para alterar qual pod voce está acessando basta alterar a parte do comando `cockroachdb-2` para o nome da pod que você deseja acessar.

#### 3.2. Dentro da pod inicialize o [build-in SQL client](https://www.cockroachlabs.com/docs/v20.2/cockroach-sql) do cockroach

```shell
$ cockroach sql --certs-dir cockroach-certs
```

```shell
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
# Server version: CockroachDB CCL v20.2.0 (x86_64-unknown-linux-gnu,
built 2020/11/09 16:01:45, go1.13.14) (same version as client)
# Cluster ID: ff66ae62-8a67-4e24-a636-ce5f5fd2607d
#
root@:26257/defaultdb>
```

A partir deste momento, já é possível executar comandos SQL diretamente em nossas aplicações do cockroachdb.

#### 3.3. Crie o banco de dados.

```sql
CREATE DATABASE commic_book
```

#### 3.4. Popular a base de dados

Agora vamos importar a nossa base antes de iniciar os testes, e para isso utilizaremos o arquivo [marvel.csv](https://raw.githubusercontent.com/bernacamargo/PMD-tutorial/using-gcloud/marvel.csv).

```sql
IMPORT TABLE commic_book.marvel (
    url STRING,
    name_alias STRING,
    appearances STRING,
    current STRING,
    gender STRING,
    probationary STRING,
    full_reserve STRING,
    years STRING,
    years_since_joining STRING,
    honorary STRING,
    death1 STRING,
    return1 STRING,
    death2 STRING,
    return2 STRING,
    death3 STRING,
    return3 STRING,
    death4 STRING,
    return4 STRING,
    death5 STRING,
    return5 STRING,
    notes STRING
)
CSV DATA ("https://raw.githubusercontent.com/bernacamargo/PMD-tutorial/using-gcloud/marvel.csv")
;
```

Caso o importe não encontre nenhum problema o retorno esperado deve ser:

            job_id       |  status   | fraction_completed | rows | index_entries | bytes
    ---------------------+-----------+--------------------+------+---------------+--------
    619735075436953602 | succeeded |                  1 |  159 |             0 | 31767
    (1 row)

    Time: 617ms total (execution 617ms / network 0ms)


Agora realize um SELECT na tabela para ver seus dados.

```sql
SELECT * FROM commic_book.marvel;
```

#
### 4. Testes 

>Nota: É importante ressaltar que temos um cluster kubernetes, composto de três instâncias de máquina virtual (1 master e 2 workers), onde as pods são alocadas e cada pod representa um nó do CockroachDB que está executando. Dessa forma quando falamos sobre os nós do cockroachdb estamos nos referindo as pods e quando falamos dos nós do cluster estamos falando das instâncias de máquina virtual do Kubernetes.


#### 4.1 Tolerância à falhas 
    
A tolerância à falhas tem como objetivo impedir que alguma mudança da nossa base de dados seja perdida por conta de algum problema, com isso é realizado o método de replicação para que todos os nós tenham as mudanças realizadas, e assim caso um nó tenha algum problema, o outro nó do sistema terá as informações consistentes. 

Sabendo disso, vamos simular alguns casos para você perceber o este funcionamento. 
Antes de simular uma falha do nó, vamos passar pelo conceito da replicação na prática, para isso vamos efeturar uma operação de atualização(UPDATE) em um nó e verificar o que acontece com os outros nós. 

<!-- >Nota: Acesse o bash de uma das pods que estão rodando a aplicação e Dentro da pod inicialize o [build-in SQL client](https://www.cockroachlabs.com/docs/v20.2/cockroach-sql) do cockroach como explicado no item 3. Executando comandos SQL na pod. -->

##### 4.1.1. Replicação de dados

Primeiramente vamos verificar como está o dado que desejamos modificar, execute o seguinte comando SQL para busca-lo na tabela `marvel`.

```sql
SELECT url, name_alias FROM commic_book.marvel WHERE url='http://marvel.wikia.com/Anthony_Stark_(Earth-616)';
```

O retorno esperado é:

                            url                        |   name_alias
    ----------------------------------------------------+-----------------
    http://marvel.wikia.com/Anthony_Stark_(Earth-616) | Homem de ferro
    (1 row)

    Time: 2ms total (execution 1ms / network 0ms)


Execute o comando abaixo para realizar a alteração no nó 2. 

```sql
UPDATE commic_book.marvel SET name_alias = ('Homem de ferro') WHERE  url='http://marvel.wikia.com/Anthony_Stark_(Earth-616)';
```

E agora acesse o nó 1 e execute a consulta novamente

```sql
SELECT url, name_alias FROM commic_book.marvel WHERE url='http://marvel.wikia.com/Anthony_Stark_(Earth-616)';
```
Como podemos observar, a atualização foi realizada e também foi replicada para as outras pods. Dessa forma podemos realizar este mesmo teste com as outras pods e veremos que todas estão sincronizadas.

##### 4.1.2 Simulando a falha de uma pod.
   
   Vamos deletar um nó do cockroachdb utilizando o comando abaixo:
   
```shell
$ kubectl delete pod cockroachdb-2
```
        
Você terá o retorno que o nó foi deletado.  

    pod "cockroachdb-2" deleted

O que é interessante, é que no nosso example.yaml, informamos que teremos 3 nós, então quando deletamos o nó 2, o Kubernets irá verificar que o nó 2 teve uma falha, e  automaticamente reiciciará o nó e atualizará os dados baseados nos outros nós.

Executando esse comando no terminal, verificamos que a pod já esta ativa novamente. 
        
```shell
$ kubectl get pod cockroachdb-2
```

    NAME            READY     STATUS    RESTARTS   AGE
    cockroachdb-2   1/1       Running   0          15s
  
#### 4.2 Escalabilidade

> Textinho explicativo da renatinha parsa

> Scale up = Aumentar numero de pods (escalonamento horizontal)

> Scale down = Diminuir numero de pods (escalonamento horizontal)

>Nota: Vale ressaltar que o cockroachdb precisa de pelo menos 3 nós para funcionar em cloud.

##### 4.2.1. Modificar o número de nós do cockroachdb

Nesta etapa iremos editar a quantidade de `nodes` que nossa aplicação do cockroachdb irá se sustentar.

Primeiramente abra o arquivo `example.yaml` com o vim

```shell
$ vim example.yaml
```

Agora altere a última linha que explicita o número de nodes da aplicação e defina para **5** o valor dos `nodes`

O arquivo alterado deve ficar da seguinte forma:

```yaml
...
tlsEnabled: true
image:
    name: cockroachdb/cockroach:v20.2.0
nodes: 5
```

Com o arquivo salvo, podemos executar o deploy da aplicação novamente com o comando

```shell
$ kubectl apply -f example.yaml
```

O retorno deve ser parecido com o seguinte:

    crdbcluster.crdb.cockroachlabs.com/cockroachdb configured

>Nota: O comando `apply` do Kubernetes permite que alteremos a configuração inicial da aplicação do cockroachdb sem que seja necessário reinicia-la.

Podemos verificar que nossa aplicação foi escalonada através das pods existentes

```shell
$ kubectl get pods
```

O retorno deve ser parecido com o seguinte:

    NAME                                  READY   STATUS    RESTARTS   AGE
    cockroach-operator-6867445556-5ll4v   1/1     Running   0          154m
    cockroachdb-0                         1/1     Running   0          152m
    cockroachdb-1                         1/1     Running   0          151m
    cockroachdb-2                         1/1     Running   0          150m
    cockroachdb-3                         1/1     Running   0          15m
    cockroachdb-4                         1/1     Running   0          15m

Dessa forma todas as requisições feitas à aplicação serão diluídas em mais dois nós (cockroachdb-3 e cockroachdb-4).

>Nota: Para realizar a redução na quantidade de nós basta refazer o procedimento explicado acima reduzindo o número de nós. 

### MemSQL
