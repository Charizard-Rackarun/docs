# Especificação de Requisitos do Sistema — VitrineIA

## 1. Requisitos Funcionais (RF)

### 1.1 Autenticação e Controle de Acesso

* **RF01 — Autenticação de Usuários:** O sistema deve permitir o login seguro de administradores do shopping, lojistas (gestores e operadores) e consumidores por meio de credenciais individuais.
* **RF02 — Controle de Acesso Baseado em Perfis (RBAC):** O sistema deve restringir o acesso às funcionalidades e aos dados de acordo com o perfil do usuário autenticado, garantindo que cada lojista visualize exclusivamente os dados operacionais do seu próprio estabelecimento e insights agregados permitidos.
* **RF03 — Recuperação de Senha:** O sistema deve disponibilizar fluxo seguro para redefinição de senha em caso de perda ou esquecimento de credenciais.

### 1.2 Gestão de Estabelecimentos e Usuários

* **RF04 — Cadastro e Gestão de Lojas:** O sistema deve permitir o cadastro, edição e inativação de estabelecimentos comerciais do shopping, classificando-os por segmento de atuação.
* **RF05 — Gestão de Equipe por Estabelecimento:** O sistema deve permitir que o gestor principal (Master) de uma loja cadastre e gerencie sub-usuários (operadores/atendentes) vinculados ao seu estabelecimento.

### 1.3 Coleta e Importação de Dados Comerciais

* **RF06 — Importação de Dados de Vendas:** O sistema deve permitir que os lojistas insiram dados comerciais manualmente ou realizem o upload estruturado de planilhas (CSV/XLSX) contendo histórico de vendas, faturamento, volume de itens e categorias comercializadas.
* **RF07 — Validação e Saneamento de Dados:** O sistema deve validar a estrutura, a tipagem e a integridade dos arquivos importados antes da persistência no banco de dados, notificando o usuário sobre eventuais inconsistências.

### 1.4 Coleta de Preferências e Engajamento de Consumidores

* **RF08 — Criação e Disponibilização de Pesquisas:** O sistema deve permitir a criação de formulários digitais rápidos de pesquisa de preferência e consumo, acessíveis aos visitantes do shopping via QR Code ou link direto.
* **RF09 — Registro de Respostas Voluntárias:** O sistema deve armazenar as respostas submetidas pelos consumidores associando-as ao perfil anonimizado ou autenticado do visitante para fins de análise comportamental.

### 1.5 Sistema de Recompensas e Fidelização

* **RF10 — Acúmulo de Pontos:** O sistema deve creditar pontos na carteira digital do consumidor mediante a conclusão de pesquisas de preferência ou interações validadas com as lojas participantes.
* **RF11 — Gestão de Ofertas e Vouchers:** O sistema deve permitir que os lojistas cadastrem recompensas, descontos ou benefícios com definição de custo em pontos, regras de uso e prazo de validade.
* **RF12 — Resgate e Baixa de Recompensas:** O sistema deve permitir que o consumidor resgate recompensas utilizando seu saldo de pontos, gerando um código único de validação para que o operador da loja realize a baixa no ato do atendimento.

### 1.6 Dashboards e Inteligência Comercial

* **RF13 — Painel de Indicadores Comerciais (Dashboard):** O sistema deve exibir métricas consolidadas de desempenho da loja, incluindo faturamento total, quantidade de vendas, ticket médio, curva de produtos mais vendidos e comparativos entre períodos.
* **RF14 — Análise Comportamental de Consumidores:** O sistema deve apresentar painéis analíticos agregados sobre interesses predominantes, categorias mais buscadas e tendências de consumo identificadas no shopping.
* **RF15 — Mapeamento de Oportunidades e Sinergias:** O sistema deve cruzar dados de consumo e segmentos para identificar oportunidades de vendas cruzadas (cross-selling) e sugerir parcerias entre lojas de segmentos complementares.

### 1.7 Inteligência Artificial e Campanhas

* **RF16 — Assistente Comercial com IA:** O sistema deve disponibilizar uma interface conversacional integrada a um modelo de inteligência artificial capaz de interpretar os indicadores da loja e o comportamento do público para responder a perguntas gerenciais e gerar recomendações estratégicas.
* **RF17 — Gestão de Campanhas Colaborativas:** O sistema deve permitir a criação, configuração e acompanhamento de campanhas comerciais individuais ou conjuntas entre lojas parceiras, com definição de público-alvo e monitoramento de conversão.

---

## 2. Requisitos Não Funcionais (RNF)

### 2.1 Segurança e Privacidade

* **RNF01 — Conformidade com a LGPD:** O tratamento, armazenamento e exibição de dados dos consumidores devem obedecer estritamente à Lei Geral de Proteção de Dados, garantindo que os lojistas acessem apenas dados comportamentais agregados e anonimizados.
* **RNF02 — Proteção de Dados e Autenticação:** Todas as senhas devem ser armazenadas utilizando algoritmos fortes de hash criptográfico, e a comunicação entre cliente e servidor deve ser autenticada via tokens seguros (JWT) sobre protocolo HTTPS.
* **RNF03 — Rastreabilidade e Auditoria:** Operações críticas, como importação de dados, alterações cadastrais e baixa de vouchers de recompensa, devem registrar o identificador do usuário responsável e o carimbo de data/hora.

### 2.2 Desempenho e Escalabilidade

* **RNF04 — Eficiência no Processamento Analítico:** O processamento de planilhas importadas e a agregação de dados analíticos realizados em Python e Pandas devem executar de forma assíncrona ou otimizada, evitando bloqueios na interface do usuário.
* **RNF05 — Tempo de Resposta da API:** As requisições padrão de consulta aos dashboards e operações de resgate de recompensas devem apresentar tempo de resposta inferior a 2 segundos em condições normais de carga.

### 2.3 Usabilidade e Compatibilidade

* **RNF06 — Design Responsivo:** A interface desenvolvida em Next.js deve adaptar-se adequadamente a diferentes resoluções de tela, garantindo usabilidade plena tanto em computadores (painel do lojista) quanto em dispositivos móveis (acesso do consumidor).
* **RNF07 — Clareza na Visualização de Dados:** Os gráficos e indicadores gerenciais implementados com Chart.js devem apresentar legendas claras, filtros intuitivos por período e hierarquia visual adequada para tomada de decisão.

### 2.4 Arquitetura e Manutenibilidade

* **RNF08 — Modularidade e Desacoplamento:** O código-fonte do backend (FastAPI) e do frontend (Next.js) deve ser estruturado em domínios e componentes modulares, facilitando a manutenção evolutiva e a adição de novas fontes de dados.
* **RNF09 — Integridade Transacional:** As operações de crédito e débito de pontos no sistema de recompensas devem ser executadas dentro de transações atômicas no PostgreSQL, impedindo inconsistências de saldo ou duplicidade de resgates.
* **RNF10 — Disponibilidade e Resiliência:** Em caso de indisponibilidade temporária da API externa de inteligência artificial generativa, os demais módulos analíticos e operacionais da plataforma devem continuar operando normalmente sem degradação crítica.
