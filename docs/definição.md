# Fluxdesk

**Sistema de Help Desk Open Source**

**v1.0.0**

## 1. Definição

O **Fluxdesk** é um sistema de Help Desk de código aberto desenvolvido para centralizar, organizar e gerenciar solicitações de suporte em um único ambiente. A plataforma pode ser utilizada por empresas, organizações e profissionais que precisam estabelecer um canal de comunicação eficiente com seus clientes e usuários.

A proposta é reunir atendimento em tempo real, gerenciamento de chamados, organização de equipes e acompanhamento de solicitações em uma única solução. O sistema oferece um chat integrado aos sites e produtos das organizações, além de ferramentas para gerenciar conversas, distribuir responsabilidades e acompanhar cada solicitação até sua resolução.

## 2. Funcionalidades

### 2.1. Chat ao vivo

O Fluxdesk disponibiliza um código de incorporação (*embed code*) que permite integrar um widget de atendimento diretamente aos sites e produtos das organizações.

Ao clicar no widget, o visitante terá acesso a uma interface de chat inspirada em aplicativos de mensageria, como o WhatsApp, por meio da qual poderá conversar com a equipe de suporte ou interagir com um chatbot, conforme os recursos habilitados pela organização.

Cada conversa poderá ser associada a uma solicitação de suporte, permitindo que o atendimento seja acompanhado e gerenciado pela equipe responsável.

### 2.2. Quadro Kanban de chamados

A equipe de suporte terá acesso a um quadro Kanban para visualizar, organizar e acompanhar as solicitações recebidas ao longo de seu ciclo de atendimento.

Cada chamado deverá conter informações como:

- **Identificação:** ID único do chamado.
- **Solicitante:** nome e endereço de e-mail do usuário.
- **Descrição:** problema relatado ou solicitação apresentada.
- **Prioridade:** nível de urgência do atendimento.
- **Prazo:** data limite prevista para resolução.
- **Status:** etapa atual do chamado no fluxo de atendimento.

O fluxo de trabalho será dividido nas seguintes etapas:

1. Novo chamado.
2. Triagem.
3. Em atendimento.
4. Em resolução técnica.
5. Resolvido.
6. Encerrado.

Os analistas poderão atualizar o status dos chamados conforme o andamento do atendimento, facilitando a identificação de pendências, a distribuição de responsabilidades e o acompanhamento das demandas.

### 2.3. Usuários e autenticação

O sistema contará com autenticação baseada em JWT (*JSON Web Token*) para permitir o acesso seguro à plataforma.

Os usuários poderão atuar como analistas de suporte, acessando as conversas e os chamados das organizações às quais estiverem vinculados, de acordo com suas permissões.

O sistema deverá controlar o acesso às informações e às operações disponíveis para cada usuário, garantindo que os recursos de atendimento e gerenciamento sejam acessíveis apenas a pessoas autorizadas.

### 2.4. Organizações e isolamento de dados

O Fluxdesk permitirá que cada usuário crie e administre sua própria organização de suporte, funcionando como administrador do ambiente criado.

Cada organização terá um espaço de trabalho independente, com seus próprios:

- Analistas e membros da equipe.
- Chamados e solicitações de suporte.
- Conversas e histórico de atendimento.
- Configurações de atendimento.
- Código de incorporação do widget de chat.

Essa estrutura permitirá que diferentes empresas ou equipes utilizem a mesma plataforma sem compartilhar indevidamente seus dados, usuários ou atendimentos.

Cada organização poderá incorporar seu próprio widget aos respectivos sites e produtos, direcionando as conversas recebidas para o ambiente de suporte correspondente

.

### 2.5. Caixa de entrada de conversas

O Fluxdesk contará com uma caixa de entrada centralizada, inspirada em sistemas de mensageria como o WhatsApp, para facilitar a comunicação entre os clientes e a equipe de suporte.

A interface será organizada em duas áreas principais:

- **Lista de conversas:** painel lateral com as conversas abertas e os atendimentos recebidos, permitindo identificar e selecionar rapidamente cada cliente.
- **Área de atendimento:** painel principal dedicado à conversa selecionada, exibindo as mensagens trocadas e permitindo que o analista responda diretamente ao cliente.

A caixa de entrada permitirá que os analistas acompanhem múltiplas conversas em um único ambiente, reduzindo a necessidade de alternar entre ferramentas e facilitando a continuidade dos atendimentos.

As conversas poderão ser associadas aos respectivos chamados, conectando a comunicação com o gerenciamento das solicitações no Kanban.

## 3. Objetivo do projeto

O objetivo do Fluxdesk é oferecer uma alternativa open source para centralizar o atendimento ao cliente e o gerenciamento de suporte técnico, combinando comunicação em tempo real, organização de chamados e gestão de equipes em uma única plataforma.

A solução busca proporcionar uma experiência simples para os usuários que solicitam suporte e uma estrutura organizada para as equipes responsáveis pelo atendimento, com possibilidade de evolução por meio de integrações, automações e novos recursos.
