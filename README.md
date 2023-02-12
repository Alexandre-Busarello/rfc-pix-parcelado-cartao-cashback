# Proposta de RFC: Parcelamento do Pix no Cartão com Cashback

- [Proposta de RFC: Parcelamento do Pix no Cartão com Cashback](#proposta-de-rfc-parcelamento-do-pix-no-cartão-com-cashback)
- [Lista de Aprovadores](#lista-de-aprovadores)
- [Resumo](#resumo)
	- [1. UI \& UX](#1-ui--ux)
	- [2. Alterações do Negócio](#2-alterações-do-negócio)
	- [2.1. Regras de Negócio](#21-regras-de-negócio)
	- [2.2. Casos de Uso](#22-casos-de-uso)
	- [3. Mudanças na Arquitetura e Fluxos do Processo](#3-mudanças-na-arquitetura-e-fluxos-do-processo)
		- [3.1. Fluxo de Pagamento por Pix Parcelado Iniciado pelo Parceiro](#31-fluxo-de-pagamento-por-pix-parcelado-iniciado-pelo-parceiro)
		- [3.2. Fluxo de Pagamento por Pix Parcelado Iniciado pelo Cliente](#32-fluxo-de-pagamento-por-pix-parcelado-iniciado-pelo-cliente)
		- [3.3. Fluxo de Escolha do Pagamento](#33-fluxo-de-escolha-do-pagamento)
		- [3.4. Fluxo de Escolha do Pagamento da Entrada ou Parcelamento 100% no Cartão](#34-fluxo-de-escolha-do-pagamento-da-entrada-ou-parcelamento-100-no-cartão)
		- [3.5. Fluxo do Pagamento com o Cartão](#35-fluxo-do-pagamento-com-o-cartão)
	- [4. Fases de Execução](#4-fases-de-execução)
	- [5. Detalhes Técnicos de Implementação](#5-detalhes-técnicos-de-implementação)
		- [5.1. Padrões de Mensagens do Barramento](#51-padrões-de-mensagens-do-barramento)
		- [5.2. Serviços de API](#52-serviços-de-api)
		- [5.3. Princípios SOLID e DDD](#53-princípios-solid-e-ddd)
		- [5.3.1. Camada de Aplicação](#531-camada-de-aplicação)
		- [5.3.2. Camada de Dominio](#532-camada-de-dominio)
		- [5.3.3. Camada de Infraestrutura](#533-camada-de-infraestrutura)
		- [5.4. Vantagens da Separação em Vários Serviços](#54-vantagens-da-separação-em-vários-serviços)
		- [5.5. Porque usar um BFF para o Frontend?](#55-porque-usar-um-bff-para-o-frontend)
	- [6. Dependências de Biblioteca](#6-dependências-de-biblioteca)
	- [7. Preocupações com segurança](#7-preocupações-com-segurança)
	- [8. Testes e Implantação](#8-testes-e-implantação)
	- [9. Acessibilidade](#9-acessibilidade)
	- [10. Referências](#10-referências)

# Lista de Aprovadores

- Equipe de Negócios
- Equipe de Arquitetura
- Equipe de UX/UI
- Equipe de Segurança

# Resumo

Este documento apresenta uma proposta para a implementação de uma solução de parcelamento do PIX utilizando cartão de crédito. O objetivo é permitir que os consumidores finais possam parcelar suas transações com PIX e receber Cashback de parceiros.

Nesta solução, será realizada uma intermediação do pagamento parcelado pelo consumidor até o recebimento final do lojista, realizado em forma de um único PIX. Além disso, a proposta inclui a utilização da plataforma `OpenPIX` para integrar as demais camadas e serviços necessários para a implementação.

## 1. UI & UX

Nossa equipe de UX/UI trabalhará para desenvolver uma interface amigável e intuitiva para que os lojistas possam oferecer o pagamento parcelado com PIX aos seus clientes. Para isso, foi criado um wireframe de baixa fidelidade (mobile first) como referência para o desenvolvimento da interface, que pode ser encontrado neste [link](https://www.figma.com/file/Jq4LodTC09wtVFhaF1pmGt/Untitled?node-id=0%3A1&t=g8s3HajlfI8J5NQB-1).

O wireframe apresenta um exemplo do fluxo para a opção de parcelamento em 2x, com uma entrada no PIX e uma parcela no cartão de crédito.

## 2. Alterações do Negócio

## 2.1. Regras de Negócio

- 1: O lojista parceiro deve poder criar seu pagamento podendo escolher habilitar o parcelamento do pix e receber um link para repassar a seu cliente;
- 2: O parcelamento do PIX deve poder ser feito em 1 entrada + 11x no cartão ou em até 12x no cartão;
- 3: Deve ser retornado o % configurado pelo parceiro como cashback para a conta do cliente em qualquer uma das opções de pagamento;
- 4: O lojista deve receber seu pagamento como se fosse um único Pix;
- 5: O parcelamento acima de 1x deve ter juros referente ao parcelamento do cartão;
- 6: O parcelamento acima de 1 no Pix + 11x no cartão deve ter uma vantagem de preço com relação as 12x no cartão;
- 7: Caso o Pix da primeira parcela seja paga mas não seja possível confirmar o pagamento das demais parcelas no cartão, o valor da entrada deve ser estornado ao usuário;

## 2.2. Casos de Uso

No diagrama abaixo, é exposto todas as ações em forma de caso de uso a serem executas por cada ator envolvido.
Foi representado 3 atores: Parceiro, Cliente e Aplicação.

- Parceiro é um lojista cadastrado;
- Cliente é um cliente cadastrado que utilizara algum lojista parceiro;
- Aplicação é o ecossistema como um todo que ira fazer os processametos em background, esses fluxos de aplicação são startados pelos casos de uso do cliente;
  - O caso de uso "Gerar cashback" extende tanto a confirmação do pagamento no cartão quanto do Pix, pois ele pode acontecer em qualquer um dos dois fluxos (mas nunca nos dois), o que acontecer por ultimo gera o cashback;
  - O mesmo pode ocorrer com os casos de usos "Estornar pagamento do Pix" e "Estornar pagamento no cartão de cŕedito", são fluxo que podem acontecer quando ocorre a recusa de um dos meios de pagamento (por isso estã com extends);
- O diagrama ficou um pouco complexo, mas representa todos os fluxos que poderiam ocorrer dentro da solução proposta para esta RFC. Ele mostra tudo que pode acontecer, mas não de maneira ordenada já que o diagrama de casos de uso não tem como objetivo definir a ordem das coisas, mas sim tudo que pode acontecer pelas ações dos atores.

<img src="/imagens/casos%20de%20uso/casos%20de%20uso.png"/>

Para não deixar o diagrama de caso de uso acima ainda mais complexo, foi omitido o caso de uso "Efetuar pagamento por Pix para o Parceiro" que ocorrerá de forma automatica pela aplicação. Esse fluxo pode acontecer tanto na confirmação do pagamento do Pix quanto na confirmação do pagamento do cartão de crédito, vai depender de quem será executado por ultimo. A única condição é que ambos precisam estar confirmados.

## 3. Mudanças na Arquitetura e Fluxos do Processo

Neste tópico, examinaremos a arquitetura proposta para atender aos requisitos do RFC, bem como todos os fluxos de pagamento possíveis, desde o início pelo Parceiro até o pagamento pelo Cliente.

Sugiro as seguintes mudanças na arquitetura:
- Modificação na API do OpenPix;
- Implementação de uma fila de mensagens usando RabbitMQ;
- Desenvolvimento de um novo Frontend para o Pix Parcelado;
- Criação de um serviço BFF para integrar o Frontend com o Backend;
- Desenvolvimento de um serviço para gerenciar pagamentos parcelados com Pix;
- Implementação de um serviço para gerenciar o saldo e cashback dos usuários.

Abaixo, encontra-se uma imagem da arquitetura proposta, baseada em um diagrama Excalidraw:

<img src="/imagens/arquitetura/ArquiteturaExcalidraw.png"/>

Neste diagrama, foi considerada uma nova arquitetura de serviços e frontend para atender exclusivamente o pagamento parcelado com Pix. A parte esquerda representa a integração com o ecossistema existente da Woovi, embora possa haver mais componentes nesta área que não foram incluídos devido à minha falta de conhecimento sobre o assunto.

### 3.1. Fluxo de Pagamento por Pix Parcelado Iniciado pelo Parceiro

<img src="/imagens/fluxos/Fluxo1.png"/>

Neste fluxo, ilustramos o início do processo de pagamento por Pix parcelado que é iniciado pelo parceiro ou lojista. Como o processo ocorre de forma síncrona, através de APIs do Serviço e de um Frontend utilizado pelo Parceiro, não temos complexidade de processamento assíncrono e mensageria.

### 3.2. Fluxo de Pagamento por Pix Parcelado Iniciado pelo Cliente

<img src="/imagens/fluxos/Fluxo2.png"/>

Neste fluxo, ilustramos o início do pagamento por Pix parcelado pelo Cliente após ele receber o link. Assim como no fluxo anterior, tudo ocorre de forma síncrona.

### 3.3. Fluxo de Escolha do Pagamento

<img src="/imagens/fluxos/Fluxo3.png"/>

Neste fluxo, ilustramos a escolha do Cliente entre pagar o valor total em uma única vez ou parcelado. Esse é o continuamento do Fluxo 3.2. Nesta etapa, temos o envolvimento de dois serviços distintos através do BFF. Embora os serviços possam ser executados de forma assíncrona pelo BFF, o retorno para o Frontend deve aguardar o fim da execução dos dois serviços envolvidos para fornecer uma resposta síncrona.

### 3.4. Fluxo de Escolha do Pagamento da Entrada ou Parcelamento 100% no Cartão

<img src="/imagens/fluxos/Fluxo4.png"/>

Neste fluxo, o Cliente tem duas opções após a conclusão do Fluxo 3.3: pagar a entrada pelo Pix e o restante no cartão ou pagar 100% no cartão sem a entrada.

Aqui temos a primeira interação com o Barramento (nosso serviço de mensageria). Devido ao fato do Cliente poder escolher pagar a entrada pelo Pix, é necessário identificar esse pagamento e confirmá-lo de forma assíncrona em um momento posterior. Além disso, precisamos envolver o novo serviço de Pix parcelado para registrar a confirmação da entrada gerada no pagamento em seu domínio e base de dados.

### 3.5. Fluxo do Pagamento com o Cartão

<img src="/imagens/fluxos/Fluxo5.png"/>

Nesta etapa, temos a maior parte dos processamentos assíncronos, pois é a ponta final do pagamento. É importante manter os serviços sincronizados através do barramento, para evitar prender o cliente na interface durante todo esse processo. Aqui, precisamos aguardar o processamento do cartão por um serviço externo (Adyen) e, em caso de sucesso, conceder o cashback e notificar o usuário via socket.

Se houver falha na etapa de pagamento com o cartão (independentemente do motivo), é importante garantir que o cliente possa fazer o estorno de pagamento por Pix. O Excalidraw ilustra boa parte dos processamentos assíncronos iniciados nesta etapa, mas ainda é importante revisar, pois pode haver algo que não foi previsto.

A utilização do barramento para mensagem (e eventos) além de tornar o processamento mais rápido, também traz uma vantagem adicional: a visibilidade fornecida pelo RabbitMQ sobre tudo que pode ter falhado, e a opção de reprocessamento de mensagens em caso de erro, garantindo a confiabilidade da aplicação.

## 4. Fases de Execução

A seguir, segue um diagrama que ilustra as etapas da implementação até a conclusão do projeto. Este diagrama não fornece detalhes sobre "como fazer", mas sim serve como uma referência para a divisão das entregas e para a execução das implementações até o final de todas as iterações. É importante destacar que essas fases não são períodos de tempo ou sprints. Isso será decidido pelo próprio time, conforme sua capacidade. É possível que uma fase seja composta por várias sprints ou que uma ou mais fases sejam realizadas em uma única sprint.

<img src="/imagens/fases/Fases.png"/>

## 5. Detalhes Técnicos de Implementação

### 5.1. Padrões de Mensagens do Barramento

Para manter uma estrutura consistente e organizada na comunicação através do barramento de mensagens, é importante seguir um padrão de esquema para as mensagens. É sugerido o seguinte formato para o esquema das mensagens:

```json
	{
	  "type": "confirm-pix-payment",
	  "data": {
	    "payment_guid": 1, 
	    "customerId": 4444,
	    "value": 1000
	  },
	  "transaction_id": "487878",
	  "timestamp": "2022-12-31T23:59:59Z"
	}
```

Aqui, o atributo `data` contém o payload real da mensagem, enquanto `type` indica o tipo de mensagem sendo enviado. Além disso, o `transaction_id` é usado para rastrear a mensagem na aplicação, e o `timestamp` indica quando a mensagem foi enviada.

Para manter a organização no barramento do RabbitMQ, é importante criar filas para cada serviço que irá utilizar o barramento para troca de mensagens. Algumas filas que devem ser criadas incluem:

- open-pix-service-queue;
- installment-pix-service-queue;
- cashback-service-queue.

Adotando esses padrões de mensagens, é possível garantir que as comunicações entre os serviços sejam claras e consistentes, facilitando a manutenção e evolução da aplicação ao longo do tempo.

### 5.2. Serviços de API

Além de estarem conectados ao barramento para trocar mensagens entre si, todos os serviços devem ter sua própria API para acessar seus recursos e operações. O serviço BFF deve oferecer uma API clara para ser usada pelo front-end, integrando aos serviços. Os serviços de domínio, como o Pix Parcelado e o Cashback, devem fornecer APIs para consultar e armazenar informações conforme necessário, para integração com o BFF.

As APIs devem ser fornecidas através do protocolo HTTP e do padrão REST. O BFF pode ter sua API implementada com GraphQL se necessário, pois ele atua como uma gateway para o front-end, mas todos os outros serviços devem fornecer suas informações através de uma API REST.

### 5.3. Princípios SOLID e DDD

As implementações dos serviços devem seguir os princípios SOLID e DDD, apresentando sempre três camadas distintas de separação de responsabilidades:

### 5.3.1. Camada de Aplicação

A camada de aplicação é responsável por intermediar entre a camada de infraestrutura e a camada de domínio. Ela processa e orquestra todas as entidades envolvidas para atender uma única responsabilidade, como, por exemplo, conceder o cashback no serviço de cashback. Esta camada não tem as regras de negócio, que são responsabilidade das classes de domínio ou repositórios. A camada de aplicação pode se comunicar com as classes de domínio, repositórios e fábricas para atender uma responsabilidade.

### 5.3.2. Camada de Dominio

A camada de domínio é onde ficam as implementações da representação do domínio na forma de classes, fábricas e repositórios. É nesta camada que está o coração da aplicação, bem como as suas regras de negócio. Aqui, utilizamos as técnicas da orientação a objetos para modelar as classes e relações de forma que faça sentido para o negócio. É importante notar que, nesta ótica, o repositório faz parte da camada de domínio e pode realizar operações de negócio em suas consultas e gravações no banco, mas não acessa diretamente o banco. Em vez disso, o acesso ao banco é abstraído por um gateway na camada de infraestrutura, que fornece acesso ao banco para realizar consultas e operações de gravação de fato.

### 5.3.3. Camada de Infraestrutura

A camada de infraestrutura é onde são abstraídos tudo que depende de aglo externo ou alguma camada de rede: como a API via protocolo HTTP, o acesso ao banco de dados, o acesso a serviços externos e as filas de mensageria. As filas e bancos de dados, são abstraídos por meio de gateways usando interfaces na camada de domínio, permitindo a inversão de dependência.

### 5.4. Vantagens da Separação em Vários Serviços

- A arquitetura proposta permitirá a separação de responsabilidades e o desacoplamento das mesmas;
- Ao adicionarmos novas funcionalidades ao processo de parcelamento do Pix, haverá um serviço específico e isolado responsável por essa tarefa. Isso significa que, caso surja algum problema ou bug (não intencional) na funcionalidade de parcelamento do Pix, apenas essa parte será afetada, mantendo todas as outras funcionalidades da OpenPix e da Woovi intactas;
- Além disso, a divisão em serviços de responsabilidade única facilitará o processo de escalabilidade da infraestrutura. Por exemplo, caso identifiquemos uma sobrecarga no uso do parcelamento do Pix, poderemos escalar apenas os serviços relacionados, sem a necessidade de escalar toda a infraestrutura que não está envolvida, o que reduzirá os custos.

### 5.5. Porque usar um BFF para o Frontend?

Mantemos um único ponto de comunicação com o frontend, responsável por agrupar e fornecer todos os dados necessários para as telas, evitando assim a necessidade de múltiplos requests a diferentes serviços no frontend.

## 6. Dependências de Biblioteca

Ao planejar as dependências de biblioteca para a implementação da solução, procuramos manter a lista de dependências o mais enxuta possível. Entretanto, foram identificadas algumas bibliotecas externas essenciais para o funcionamento da solução, as quais listamos a seguir:

- `amqplib`: é utilizada para a comunicação com o barramento, permitindo a leitura e envio de mensagens nas filas;
- `express`: é utilizado para a execução do servidor HTTP, que expõe as rotas de API de cada serviço;
- `mongoose`: é utilizado para a conexão com o banco de dados MongoDB;
- `react`: é utilizado para a implementação do frontend;
- `axios`: é utilizado para a comunicação via APIs HTTP;
- `eslint` e `prettier`: são utilizados para a configuração da padronização do código.

Observamos ainda que pode ser útil considerar a utilização do `Apollo` caso seja necessário adotar o GraphQL em algum dos serviços.

## 7. Preocupações com segurança

A segurança das informações financeiras dos usuários é uma questão crítica. É importante destacar que o setor financeiro é um alvo frequente de ataques cibernéticos, por isso é importante implementar medidas de segurança efetivas para proteger as informações sensíveis dos usuários.

Nesta seção, descrevemos as medidas de segurança que serão implementadas na solução de crédito no Pix:

- Criptografia de ponta a ponta com duas chaves (pública e privada) para garantir a confidencialidade das informações financeiras relacionadas ao cartão de crédito. A criptografia de ponta a ponta é uma técnica comprovada que protege as informações financeiras dos usuários contra intercepções não autorizadas;
- Autenticação forte dos usuários para evitar acessos não autorizados. A autenticação forte deve incluir senhas complexas, autenticação de dois fatores e outras medidas para garantir que apenas usuários autorizados tenham acesso às informações financeiras;
- Monitoramento constante do sistema para detectar e responder a ameaças. É importante monitorar constantemente o sistema para detectar ameaças potenciais, tais como ataques cibernéticos, violações de segurança ou outras ameaças. É essencial ter um plano de resposta em caso de ameaças para garantir que a segurança das informações financeiras dos usuários seja mantida.

Além destas medidas, é importante também considerar a implementação de testes de penetração para identificar pontos fracos na segurança do sistema e garantir a integridade das informações financeiras dos usuários.

## 8. Testes e Implantação

- É importante desenvolver testes unitários para assegurar que as regras de negócio estejam sendo implementadas corretamente em cada serviço
- Além disso, recomendo a criação de testes de integração para verificar se as diferentes camadas do sistema estão se comunicando corretamente

## 9. Acessibilidade

A solução UX/UI de parcelamento para o Pix deverá ser desenvolvida com acessibilidade em mente, a fim de garantir que todos os usuários, incluindo aqueles com necessidades especiais, possam usar o sistema de forma fácil e intuitiva. É importante levar em consideração a inclusão digital e garantir que a solução seja acessível para todos.

## 10. Referências

Abaixo segue o Figma dos Wireframes e o Excalidraw do fluxo completo que atender este RFC:

- Figma: https://www.figma.com/file/Jq4LodTC09wtVFhaF1pmGt/Untitled?node-id=0%3A1&t=g8s3HajlfI8J5NQB-1
- Excalidraw: https://excalidraw.com/#json=KsBqzyBfjdha47I6mHvLJ,_xcsA9sG6Dob_fHdel7B5g