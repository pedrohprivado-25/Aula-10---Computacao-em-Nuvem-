# Aula 10 – Computação em Nuvem | Escala e Disponibilidade

## Introdução

Nesta aula, aprendemos como o Kubernetes consegue manter uma aplicação funcionando mesmo quando precisamos aumentar ou diminuir a quantidade de Pods.

Durante a atividade, utilizamos o Nginx como exemplo e criamos um Service para facilitar o acesso à aplicação. Também fizemos testes de comunicação, aumentamos a quantidade de réplicas, excluímos um Pod para observar o que aconteceria e, por último, reduzimos novamente a quantidade de réplicas.

Depois, realizamos um desafio prático para observar os resultados e responder algumas perguntas sobre o funcionamento do Kubernetes.

---

# Parte 1 – Atividade Prática

## PASSO 1 – Criando o namespace e a aplicação

Primeiro, tentamos verificar os Nodes do Kubernetes:

```bash
~kubectl get nodes
```

O terminal apresentou a seguinte mensagem:

```text
~kubectl: command not found
```

Isso aconteceu porque digitamos o comando com um caractere `~` no início.

Depois, criamos o namespace `aula10`:

```bash
kubectl create namespace aula10
```

**Resultado:**

```text
namespace/aula10 created
```

Em seguida, criamos uma aplicação utilizando a imagem do Nginx:

```bash
kubectl -n aula10 create deployment web --image=nginx:alpine
```

**Resultado:**

```text
deployment.apps/web created
```

Depois, verificamos se a aplicação havia iniciado corretamente:

```bash
kubectl -n aula10 rollout status deployment/web
```

**Resultado:**

```text
deployment "web" successfully rolled out
```

Por último, consultamos o Deployment e os Pods:

```bash
kubectl -n aula10 get deployment,pods
```

**Resultado:**

```text
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web   1/1     1            1           98s

NAME                       READY   STATUS    RESTARTS   AGE
pod/web-6bd469df5c-dd5bt   1/1     Running   0          98s
```

**O que fizemos:** Criamos o ambiente da atividade e iniciamos uma aplicação Nginx com apenas uma réplica. Também verificamos se ela estava funcionando corretamente.

## PASSO 2 – Criando o Service

Depois de criar a aplicação, precisamos permitir que outros Pods pudessem acessá-la. Para isso, criamos um Service chamado `web-svc`:

```bash
kubectl -n aula10 expose deployment web --name=web-svc --port=80 --target-port=80
```

**Resultado:**

```text
service/web-svc exposed
```

Em seguida, verificamos se o Service havia sido criado:

```bash
kubectl -n aula10 get service web-svc
```

**Resultado:**

```text
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
web-svc   ClusterIP   10.103.149.209   <none>        80/TCP    2m21s
```

Também verificamos os Pods da aplicação:

```bash
kubectl -n aula10 get pods -l app=web
```

**Resultado:**

```text
NAME                   READY   STATUS    RESTARTS   AGE
web-6bd469df5c-dd5bt   1/1     Running   0          5m37s
```

**O que fizemos:** Criamos o Service para facilitar o acesso à aplicação. Ele recebeu o nome `web-svc` e foi configurado como `ClusterIP`, permitindo a comunicação dentro do próprio cluster.

## PASSO 3 – Criando um Pod para testar o acesso

Agora que tínhamos a aplicação e o Service, criamos outro Pod chamado `cliente`, utilizando a imagem BusyBox:

```bash
kubectl -n aula10 run cliente --image=busybox:1.36 --restart=Never --command -- sleep 3600
```

**Resultado:**

```text
pod/cliente created
```

Depois, esperamos até que o Pod estivesse pronto:

```bash
kubectl -n aula10 wait --for=condition=Ready pod/cliente --timeout=90s
```

**Resultado:**

```text
pod/cliente condition met
```

**O que fizemos:** Criamos um Pod separado para testar se outro Pod conseguia acessar a aplicação através do Service.

## PASSO 4 – Testando a comunicação com o Service

Com o Pod cliente pronto, fizemos uma requisição para acessar o Nginx através do nome `web-svc`:

```bash
kubectl -n aula10 exec cliente --   wget -qO- http://web-svc | head -n 8
```

**Resultado:**

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
```

Em seguida, verificamos novamente o Deployment:

```bash
kubectl -n aula10 get deployment web
```

**Resultado:**

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    1/1     1            1           9m23s
```

**O que fizemos:** Conseguimos acessar a página inicial do Nginx utilizando o Service. Isso mostrou que o Pod `cliente` estava conseguindo se comunicar com a aplicação pelo nome `web-svc`.

## PASSO 5 – Aumentando a quantidade de Pods

Depois de confirmar que o acesso estava funcionando, aumentamos a quantidade de réplicas da aplicação de 1 para 3:

```bash
kubectl -n aula10 scale deployment/web --replicas=3
```

**Resultado:**

```text
deployment.apps/web scaled
```

Em seguida, verificamos se a alteração havia terminado:

```bash
kubectl -n aula10 rollout status deployment/web
```

**Resultado:**

```text
deployment "web" successfully rolled out
```

Depois, consultamos o Deployment:

```bash
kubectl -n aula10 get deployment web
```

**Resultado:**

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    3/3     3            3           11m
```

Por último, verificamos os três Pods e seus endereços:

```bash
kubectl -n aula10 get pods -l app=web -o wide
```

**Resultado:**

```text
NAME                   READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
web-6bd469df5c-dd5bt   1/1     Running   0          11m   192.168.1.252   node01   <none>           <none>
web-6bd469df5c-dvr2v   1/1     Running   0          84s   192.168.1.1     node01   <none>           <none>
web-6bd469df5c-s2lmf   1/1     Running   0          84s   192.168.1.7     node01   <none>           <none>
```

**O que fizemos:** Aumentamos a aplicação para três réplicas. O Kubernetes criou mais dois Pods e os três apareceram como funcionando normalmente.

## PASSO 6 – Verificando o Service e os Endpoints

Depois de aumentar a quantidade de Pods, verificamos novamente o Service:

```bash
kubectl -n aula10 get service web-svc
```

**Resultado:**

```text
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
web-svc   ClusterIP   10.103.149.209   <none>        80/TCP    9m59s
```

Em seguida, consultamos os Endpoints:

```bash
kubectl -n aula10 get endpoints web-svc
```

**Resultado:**

```text
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME      ENDPOINTS                                        AGE
web-svc   192.168.1.1:80,192.168.1.252:80,192.168.1.7:80   11m
```

Depois, fizemos novamente o teste de acesso pelo Pod cliente:

```bash
kubectl -n aula10 exec cliente -- wget -qO- http://web-svc | head -n 8
```

**Resultado:**

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
```

**O que fizemos:** Verificamos que o Service continuava com o mesmo nome e endereço interno, mesmo depois do aumento das réplicas. Também observamos que os Endpoints apresentavam os endereços dos três Pods.

Conseguimos acessar o Nginx normalmente pelo Service.

Também apareceu um aviso informando que o recurso Endpoints está em desuso nas versões mais recentes do Kubernetes, sendo recomendado utilizar o EndpointSlice.

## PASSO 7 – Excluindo um Pod e acompanhando sua substituição

Agora, fizemos um teste para observar o que aconteceria se um dos Pods da aplicação fosse excluído.

Primeiro, selecionamos um dos Pods:

```bash
POD=$(kubectl -n aula10 get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
```

Depois, excluímos o Pod selecionado:

```bash
kubectl -n aula10 delete pod "$POD"
```

**Resultado:**

```text
pod "web-6bd469df5c-dd5bt" deleted from aula10 namespace
```

Em seguida, acompanhamos os Pods:

```bash
kubectl -n aula10 get pods -l app=web -w
```

**Resultado:**

```text
NAME                   READY   STATUS    RESTARTS   AGE
web-6bd469df5c-dvr2v   1/1     Running   0          6m6s
web-6bd469df5c-rbnk4   1/1     Running   0          29s
web-6bd469df5c-s2lmf   1/1     Running   0          6m6s
```

**O que fizemos:** Excluímos um dos Pods e acompanhamos o que aconteceu. Como o Deployment estava configurado para manter três réplicas, o Kubernetes criou outro Pod para substituir o que foi removido.

Assim, conseguimos perceber que o Kubernetes procura manter a quantidade de réplicas que definimos.

## PASSO 8 – Reduzindo as réplicas e finalizando a atividade

Depois dos testes, reduzimos a quantidade de réplicas de 3 para 2:

```bash
kubectl -n aula10 scale deployment/web --replicas=2
```

**Resultado:**

```text
deployment.apps/web scaled
```

Em seguida, verificamos se a alteração havia sido concluída:

```bash
kubectl -n aula10 rollout status deployment/web
```

**Resultado:**

```text
deployment "web" successfully rolled out
```

Depois, consultamos o Deployment e os Pods:

```bash
kubectl -n aula10 get deployment,pods
```

**Resultado:**

```text
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web   2/2     2            2           19m

NAME                       READY   STATUS    RESTARTS   AGE
pod/cliente                1/1     Running   0          12m
pod/web-6bd469df5c-dvr2v   1/1     Running   0          8m57s
pod/web-6bd469df5c-s2lmf   1/1     Running   0          8m57s
```

Também verificamos os Endpoints do Service:

```bash
kubectl -n aula10 get endpoints web-svc
```

**Resultado:**

```text
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME      ENDPOINTS                       AGE
web-svc   192.168.1.1:80,192.168.1.7:80   17m
```

Por último, removemos o namespace utilizado na atividade:

```bash
kubectl delete namespace aula10
```

**Resultado:**

```text
namespace "aula10" deleted
```

Depois, tentamos verificar se o namespace havia sido removido:

```bash
kubectl get namespace aula10
```

**Resultado registrado no terminal:**

```text
~kubectl: command not found
```

**O que fizemos:** Reduzimos a quantidade de réplicas para duas e verificamos que o Deployment ficou com `READY 2/2`. Também observamos que o Service passou a apresentar somente os dois Endpoints restantes.

Depois, excluímos o namespace `aula10`, encerrando a atividade. Na última verificação, o comando foi digitado novamente com `~` no início e retornou `command not found`. Por isso, essa tentativa não confirmou a consulta, embora a mensagem anterior já tivesse informado que o namespace foi excluído.

---

# Parte 2 – Desafio Prático

Depois de finalizar a atividade, utilizamos os resultados do terminal para preencher a tabela de observação e responder às perguntas propostas pela professora.

## 1. Tabela de observação

### Antes – Aplicação com uma réplica

| Observação | Resultado |
|---|---|
| Desejado | 1 |
| Pods prontos | 1 |
| Service responde? | Sim |

**O que observamos:**

No início, a aplicação estava funcionando com apenas um Pod. Fizemos um teste utilizando o Pod cliente e conseguimos acessar a página inicial do Nginx através do Service `web-svc`.

### Pico – Aplicação com três réplicas

| Observação | Resultado |
|---|---|
| Desejado | 3 |
| Pods prontos | 3 |
| Endereços | 192.168.1.1, 192.168.1.252 e 192.168.1.7 |

**O que observamos:**

Aumentamos a quantidade de réplicas de uma para três. Depois, verificamos que os três Pods estavam funcionando e que seus endereços apareciam nos Endpoints do Service.

Também fizemos outro teste de acesso e conseguimos receber novamente a página do Nginx.

### Após falha – Exclusão de um Pod

| Observação | Resultado |
|---|---|
| Pod apagado | web-6bd469df5c-dd5bt |
| Reposição | Sim, um novo Pod foi criado |
| Service responde? | Sim, os Pods e Endpoints continuaram disponíveis |

**O que observamos:**

Excluímos um dos Pods da aplicação e acompanhamos o que aconteceu. O Kubernetes criou outro Pod para substituir o que havia sido removido.

Depois, verificamos que a aplicação continuava com três Pods funcionando. Isso mostrou que o Deployment conseguiu manter a quantidade de réplicas configurada.

A disponibilidade do Service foi observada pela permanência dos Pods e Endpoints. O teste de acesso pelo cliente havia sido realizado antes dessa exclusão.

## 2. Análise em dupla

### 1. Quem repôs o Pod apagado?

Quem fez a reposição foi o Deployment, por meio do ReplicaSet que ele controla. Como a aplicação estava configurada para ter três réplicas, o Kubernetes percebeu que havia um Pod a menos e criou outro para manter a quantidade desejada.

### 2. Por que o cliente continuou usando o nome web-svc?

Porque o Service mantém um nome e um endereço de acesso estáveis dentro do cluster. Mesmo quando os Pods são substituídos, o cliente não precisa descobrir o endereço de cada Pod novamente. Ele continua acessando a aplicação pelo nome `web-svc`.

### 3. Três réplicas garantem capacidade infinita? Explique.

Não. Três réplicas ajudam a distribuir o acesso e manter mais instâncias da aplicação disponíveis, mas não garantem capacidade infinita. Cada Pod utiliza recursos do cluster, como memória e processamento. Se houver muitas requisições ao mesmo tempo, os recursos disponíveis podem não ser suficientes.

### 4. Que informação seria necessária para automatizar a escala?

Seria necessário acompanhar informações sobre o uso da aplicação, principalmente o consumo de CPU ou memória. Com essas informações, seria possível definir quando aumentar ou diminuir a quantidade de réplicas automaticamente.

Na aula, vimos que esse tipo de ajuste automático pode ser feito com o HPA, desde que a infraestrutura tenha os recursos necessários para acompanhar as métricas.

## O que aprendemos com o desafio?

Com esse desafio, conseguimos entender melhor como o Kubernetes mantém a aplicação funcionando mesmo quando um dos Pods é excluído.

Também aprendemos que o Deployment trabalha para manter a quantidade de réplicas desejada e que o Service facilita o acesso à aplicação, pois mantém o mesmo nome mesmo quando os Pods mudam.

Outro ponto importante foi perceber que aumentar a quantidade de réplicas ajuda a atender mais acessos, mas não significa que a aplicação terá capacidade ilimitada.

---

# O que aprendemos na Aula 10?

Nesta aula, aprendemos principalmente sobre escala e disponibilidade no Kubernetes.

Entendemos que podemos aumentar ou diminuir a quantidade de Pods de acordo com a necessidade da aplicação. Também vimos que o Deployment controla a quantidade desejada e que o ReplicaSet ajuda a manter essa quantidade.

Outro aprendizado foi sobre o Service, que permite acessar a aplicação por um nome fixo, mesmo quando os Pods são substituídos ou recebem novos endereços.

Também conhecemos a diferença entre escala manual e automática. Na prática, fizemos o ajuste manualmente com o comando `kubectl scale`. Já a escala automática pode ser realizada com o HPA, utilizando métricas da aplicação.

# Resumo da Atividade Prática e do Desafio

Na atividade prática, criamos uma aplicação Nginx com uma réplica e configuramos um Service chamado `web-svc` para permitir o acesso pelo Pod cliente.

Depois, testamos a comunicação, aumentamos a aplicação para três réplicas, verificamos os endereços dos Pods e dos Endpoints e excluímos um dos Pods para observar sua substituição automática pelo Kubernetes.

Em seguida, reduzimos a aplicação para duas réplicas e removemos o namespace utilizado no laboratório.

No desafio prático, utilizamos os resultados dessa atividade para preencher a tabela de observação e responder às perguntas sobre reposição de Pods, funcionamento do Service, capacidade das réplicas e escala automática.

## Conclusão

A Aula 10 foi importante para entender melhor como o Kubernetes ajuda a manter uma aplicação disponível e como podemos ajustar a quantidade de Pods conforme a necessidade.

O que mais chamou minha atenção foi perceber que, mesmo depois de excluir um Pod, o Kubernetes conseguiu criar outro para manter a quantidade de réplicas configurada.

Também consegui entender melhor a função do Service, que facilita o acesso à aplicação sem precisar acompanhar diretamente o endereço de cada Pod.

Com a atividade prática e o desafio, consegui visualizar melhor como funcionam a escala e a disponibilidade dentro do Kubernetes.

**Computação em Nuvem – Aula 10 | Escala e Disponibilidade**
