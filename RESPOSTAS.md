# A1 — Sistemas Distribuídos & Mobile — MaréPay

**Nome:** Livia de Queiroz Schendroski
**RA:** 824217515
**Nome:** Leticia Augusto da Silva
**RA:** 8261108611

![Arquitetura do MaréPay](arquitetura.png)

## Questão 1 — Para que serve cada serviço

> Para cada serviço da AWS que aparece na sua figura, explique para que
> ele serve e o que ele faz no MaréPay.

**Resposta:**

### Amazon VPC
- **Para que serve:** cria uma rede privada e isolada dentro da AWS, dividida em sub-redes (cada uma em uma zona de disponibilidade). Todos os recursos ficam dentro dela.
- **O que faz no MaréPay:** é a rede onde ficam o balanceador, os servidores e o banco. As sub-redes em duas zonas permitem que o sistema continue funcionando se uma zona falhar.

### Application Load Balancer (ALB)
- **Para que serve:** é a porta de entrada única do sistema. Recebe os pedidos, distribui entre os servidores e testa a saúde de cada um (health check), enviando pedidos só para os que estão bem.
- **O que faz no MaréPay:** recebe as compras enviadas pelas leitoras das barracas e as requisições do site, e as distribui entre os servidores das duas zonas. Se um servidor falhar, deixa de enviar pedidos a ele.

### Amazon EC2
- **Para que serve:** é um computador virtual alugado, onde se roda um programa.
- **O que faz no MaréPay:** roda a API que recebe as compras das leitoras, valida os dados e grava no DynamoDB, além de consultar no RDS o cadastro de barracas e produtos. A API não guarda dados na memória (é stateless), então qualquer servidor pode atender qualquer pedido.

### EC2 Auto Scaling
- **Para que serve:** mantém a quantidade certa de servidores, criando e removendo máquinas automaticamente a partir de um modelo (launch template).
- **O que faz no MaréPay:** mantém no mínimo 2 servidores em 2 zonas e recria qualquer um que caia (autorrecuperação). No pico do fim dos shows, cria mais servidores até o máximo definido, e depois reduz para economizar.

### Amazon DynamoDB
- **Para que serve:** é um banco de dados NoSQL, serverless, muito rápido para gravar e consultar por chave, que escala sozinho e replica os dados em várias zonas.
- **O que faz no MaréPay:** guarda todas as compras feitas com as pulseiras. A chave de partição é o id da pulseira e a de ordenação é a data e hora da compra, o que permite gravar rápido e consultar as compras de uma pulseira em ordem.

### Amazon RDS
- **Para que serve:** é um banco de dados relacional (SQL) gerenciado: a AWS cuida de instalação, atualizações e backups.
- **O que faz no MaréPay:** guarda o cadastro das barracas e dos produtos com seus preços. Com o Multi-AZ ligado, mantém uma cópia em outra zona e troca sozinho se a principal falhar.

### Amazon S3
- **Para que serve:** guarda arquivos (objetos) em buckets, com alta durabilidade e sem servidor para gerenciar. Pode hospedar sites estáticos.
- **O que faz no MaréPay:** hospeda o site simples com o mapa do festival, que o público abre no celular, e guarda o código da API em um bucket privado, de onde os servidores novos o baixam ao ligar.

### Amazon CloudWatch
- **Para que serve:** coleta métricas (números medidos ao longo do tempo), mostra em painéis e dispara alarmes quando um valor passa de um limite.
- **O que faz no MaréPay:** acompanha, por exemplo, o número de pedidos por minuto e a saúde dos servidores. Também fornece a métrica usada pelo Auto Scaling. Quando algo sai do normal, o alarme dispara.

### Amazon SNS
- **Para que serve:** serviço de avisos por tópicos: o alarme publica em um tópico e todos os inscritos recebem a mensagem.
- **O que faz no MaréPay:** envia o e-mail para a equipe quando o alarme do CloudWatch dispara.

### AWS IAM
- **Para que serve:** controla quem pode fazer o quê na AWS, por meio de permissões, seguindo o princípio do menor privilégio.
- **O que faz no MaréPay:** dá aos servidores um papel (role) com credenciais temporárias, permitindo gravar no DynamoDB e baixar o código do S3, sem guardar senha ou chave no código.

## Questão 2 — Dois bancos de dados

> O MaréPay usa o DynamoDB e o RDS. O que fica guardado em cada um?
> Por que usar dois bancos diferentes?

**Resposta:**
No **DynamoDB** ficam as **compras** feitas com as pulseiras: são muitas, chegam em grande volume e em alta velocidade (principalmente no pico), quase não mudam depois de gravadas e são buscadas por chave (id da pulseira e data e hora).

No **RDS** ficam o **cadastro das barracas e dos produtos com seus preços**: é um volume pequeno, bem estruturado e com relação entre os dados (uma barraca tem vários produtos), o que combina com tabelas e SQL.

Usam-se dois bancos porque os dados têm perfis diferentes, e cada banco é o melhor para um deles. O DynamoDB escala sozinho e grava muito rápido, ideal para as compras. O RDS oferece relações e consultas flexíveis, ideal para o cadastro. Essa prática chama-se persistência poliglota: usar o banco certo para cada tipo de dado.

## Questão 3 — Servidor caindo e pico de compras

> O que acontece no MaréPay se um servidor cair? E quando um show termina
> e muita gente compra ao mesmo tempo? Diga quais serviços resolvem cada
> um desses problemas.

**Resposta:**
**Se um servidor cair:** o **ALB** faz o health check, percebe que o servidor não responde e deixa de enviar pedidos a ele, enquanto o outro servidor (em outra zona) continua atendendo. Em seguida, o **EC2 Auto Scaling** vê que há menos servidores que o desejado e cria um novo a partir do launch template. Assim, as leituras continuam sendo processadas sem intervenção manual (autorrecuperação). Como a API é stateless e os dados ficam no DynamoDB e no RDS, nada se perde. Serviços envolvidos: **ALB, EC2 Auto Scaling, EC2** e, para os dados, **DynamoDB e RDS**.

**No pico de compras após um show:** o número de pedidos por servidor aumenta. O **CloudWatch** mede essa carga e o **EC2 Auto Scaling** (por target tracking, com meta de pedidos por servidor) cria mais servidores, até o máximo definido, e o **ALB** distribui os pedidos entre todos, inclusive os novos. O **DynamoDB** absorve as gravações sem precisar de ajuste. Quando o movimento diminui, o Auto Scaling reduz a quantidade de servidores. Se algo passar do limite, o **CloudWatch** dispara o alarme e o **SNS** avisa a equipe por e-mail. Serviços envolvidos: **CloudWatch, EC2 Auto Scaling, ALB, DynamoDB e SNS**.

## Uso de Inteligência Artificial

> Para cada IA que você usou (ChatGPT, Gemini, Claude, Copilot etc.),
> diga em que parte da prova usou e como usou (o que você pediu).
> Se não usou nenhuma, escreva "Não utilizei IA nesta prova."

### Claude (Anthropic)
- **Onde usei:** na preparação para a prova e na redação das respostas das Questões 1, 2 e 3 deste documento e figuras do diagrama.
- **Como usei (o que pedi):** enviei a apostila do AgroSensor e pedi que explicasse a arquitetura e os conceitos, com perguntas prováveis. Pedi um material de estudo em linguagem simples. Depois pedi a explicação de como montar uma arquitetura para o caso MaréPay e, contextualizei a arquitetura do MaréPay, pedi para complementar caso não estivesse com todas as ferramentas necessária. Por fim, pedi que me explicasse e preenchesse este modelo de prova com as respostas.

### Gemini (IA do Google)
- **Onde usei:** Draw.io para criar o arquivo arquitetura.png
- **Como usei (o que pedi):** Pesquisei no Google formas mais simples de centralizar e ajustar tamanhos no Draw.io e a IA de pesquisa forneceu dicas.