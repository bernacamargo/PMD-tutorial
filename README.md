# NewSQL - Tolerância à falha e escalabilidade com Cockroachdb e MemSQL

### Autores
- Bernardo Pinheiro Camargo [@bernacamargo](https://github.com/bernacamargo)
- Renata Praisler [@RenataPraisler](https://github.com/RenataPraisler)

# 

### Objetivos
No contexto de bancos de dados relacionais e distribuídos (NewSQL), temos como objetivo deste projeto planejar e elaborar um tutorial intuitivo que permita a qualquer pessoa interessada testar e validar as características relacionadas a tolerância às falhas e escalabilidade na estrutura de NewSQL.

### Introdução

O NewSQL surgiu como uma nova proposta, pois com o uso do NOSQL acabou apresentando alguns problemas como por exemplo: a falta, do uso de transações, das consultas SQL e a estrutura complexa por não ter uma modelagem estruturada. Ele veio com o objetivo de ter os os pontos positivos dos do modelo relacional para as arquiteturas distribuídas e aumentar o desempenhos das queries de SQL, não tendo a necessidade de servidores mais potentes para melhor execução, e utilizando a escalabilidade horizontal e mantendo as propriedades ACID(Atomicidade, Consistência, Isolamento e Durabilidade).


### Tecnologias Habilitadores

- Kubernetes;
- Docker;
- Google Kubernetes Engine (GKE);
- Cockroachdb;
- MemSQL;

## 

### Cluster Kubernetes

Para podermos simular um ambiente isolado e que garanta as características de sistemas distribuídos utilizaremos um cluster local orquestrado pelo Kubernetes, o qual é responsável por gerenciar instâncias de máquinas virtuais para execução de aplicativos em containers. 

Primeiramente precisamos criar nosso cluster e utilizaremos o GKE para isto:

- Acesse a [Google Cloud Console](https://console.cloud.google.com)
- Navegue até o `Kubernetes Engine` e clique em `Clusters`;
- Clique em `Criar cluster` no centro da janela;
- Defina o nome do cluster e clique em `Criar`.

Feito isso, um cluster com três nós será criado e inicializado. Em alguns momentos você já poderá acessá-lo para seguirmos com as configurações.


> Para ambos os softwares Cockroachdb e MemSQL utilizaremos o mesmo processo para inicialização do cluster kubernetes, porém em clusters diferentes.
### Cockroachdb

Antes de iniciar o teste para identificar como é realizado a tolerância a falhas e a escalabilidade, temos que configurar o Cockroachdb no nosso cluster, para nos auxiliar utilizamos as documentações do Cockroachdb e kubernetes, e citaremos abaixo os comandos que devem ser realizados.

#### 1. Iniciar o Operator do nosso cluster.

> Nota: O operator,  será responsável por manter a operação do cockroachdb dentro do nosso cluster, assim toda vez que a gente for iniciar o nó (pods), ele irá instalar tudo que é necessário dentro deste nó (pods)  

  1.1. Aplicar o CustomResourceDefinition (CRD) do Operator(operador)
  
    $ kubectl apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/master/config/crd/bases/crdb.cockroachlabs.com_crdbclusters.yaml

O retorno esperado é:

    customresourcedefinition.apiextensions.k8s.io/crdbclusters.crdb.cockroachlabs.com created

  1.2. Aplicar o Operator manifest
  
    $ kubectl apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/master/manifests/operator.yaml
    
O retorno esperado é:

    clusterrole.rbac.authorization.k8s.io/cockroach-operator-role created
    serviceaccount/cockroach-operator-default created
    clusterrolebinding.rbac.authorization.k8s.io/cockroach-operator-default created
    deployment.apps/cockroach-operator created

  1.3. Validar se o Operator está rodando
  
    $ kubectl get pods
    
  Caso tenha funcionado, você deverá ter como retorno a seguinte mensagem:

    NAME                                 READY   STATUS    RESTARTS   AGE
    cockroach-operator-6867445556-9x6zp   1/1    Running      0      2m51s

> Nota: Caso o status da pod estiver como "ContainerCreating" é só aguardar alguns instantes que o kubernetes esta iniciando o container e logo deverá aparecer como "Running".

  #### 2. Configuração em si do cluster.
  
  2.1. Realize o download e a edição do arquivo `example.yaml`, este é responsável por realizar a configuração do cluster Kubernetes.
  
    $ curl -O https://raw.githubusercontent.com/cockroachdb/cockroach-operator/master/examples/example.yaml

Utilize o `vim`(ou qualquer editor de texto de sua preferência) para abrir o arquivo baixado no passo anterior.

    $ vim example.yaml
  
  2.2. Neste passo vamos configurar nossa CPU e memoria para cada pod executada dentro do nosso cluster. Essa etapa é opicional e é recomendada em ambientes de produção, visto que é isso que irá limitar os recursos que poderão ser utilizados pela pod.
  
    resources:
        requests:
            cpu: "2"
            memory: "8Gi"
            
        limits:
            cpu: "2"
            memory: "8Gi"

> Nota: Caso não seja definido valores iniciais a aplicação poderá utilizar totalmente os recursos disponibilizados pelo host e ocasionar cobranças indesejadas.
        
   2.3. Modifique o resources.requests.storage
   
     resources:
        requests:
            storage: "1Gi"

> Nota: caso você queira outra configuração para outro teste ou projeto, se atente de verificar as configurações necessárias disponibilizadas [aqui](https://www.cockroachlabs.com/docs/v20.2/recommended-production-settings#hardware)


2.4. Aplique as configurações feitas no arquivo `example.yaml`.

    $ kubectl apply -f example.yaml

O retorno esperado é:

    crdbcluster.crdb.cockroachlabs.com/cockroachdb created    

> Nota: Este arquivo irá solicitar para o Operador que crie uma aplicação StatefulSet com três pods que funcionarão como um cluster cockroachdb.

2.5. Verifique se as pods subiram.

    $ kubectl get pods

O retorno esperado é:

    NAME                                  READY   STATUS    RESTARTS   AGE
    cockroach-operator-6867445556-9x6zp   1/1     Running   0          43m
    cockroachdb-0                         1/1     Running   0          2m29s
    cockroachdb-1                         1/1     Running   0          104s
    cockroachdb-2                         1/1     Running   0          67sa
    

#### 3. Popular nossa base de dados.
Após o sucesso das operações acima, vamos popular a nossa base de dados, para isso utilizamos um arquivo csv disponibilizado no git do [@tmcnab](https://github.com/tmcnab/northwind-mongo/blob/master/employees.csv).

3.1. 



### MemSQL
