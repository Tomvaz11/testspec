# Blueprint Arquitetural: IABANK (Tocrisna v7.5)

Este documento define a arquitetura de alto nível, técnica e de produto, para o sistema de gestão de empréstimos IABANK. Ele serve como a fonte única da verdade para a estrutura, componentes e contratos da plataforma.

## 1. Visão Geral da Arquitetura

A arquitetura escolhida é uma **Arquitetura em Camadas (Layered Architecture)** aplicada a uma aplicação **Monolítica Modular**. O backend Django será estruturado seguindo princípios da Arquitetura Limpa (Clean Architecture) para garantir uma forte separação de responsabilidades, testabilidade e manutenibilidade.

- **Monolito Modular:** Em vez de microserviços, optamos por um único codebase (monolito) dividido em módulos de negócio bem definidos (apps Django como `loans`, `finance`, `customers`). Esta abordagem reduz a complexidade operacional inicial, acelera o desenvolvimento e é ideal para a fase atual do projeto, enquanto a modularidade interna facilita uma futura extração para microserviços, se necessário.
- **Arquitetura em Camadas (Backend):** O backend será dividido em quatro camadas distintas:
  1.  **Apresentação (Presentation):** Responsável por lidar com requisições HTTP e serialização de dados. (Django REST Framework: Views, Serializers, Routers).
  2.  **Aplicação (Application):** Orquestra os casos de uso do sistema. Contém a lógica de aplicação, chama serviços de domínio e de infraestrutura. (Camada de Serviços).
  3.  **Domínio (Domain):** O coração do sistema. Contém a lógica e as regras de negócio puras. No contexto pragmático do Django, esta camada se manifesta de duas formas:
      - **Lógica Simples (no Model/Manager):** Para regras de negócio simples e autocontidas que operam sobre os dados de um único modelo (ex: um método `@property` para verificar se um empréstimo está vencido), é aceitável e idiomático usar **métodos de negócio dentro dos Models e Managers** do Django.
      - **Lógica Complexa (em Serviços/Entidades Puras):** Para orquestrações complexas que envolvem múltiplos modelos, cálculos financeiros críticos ou interações que devem ser independentes da persistência, a lógica deve residir na **Camada de Aplicação (Serviços)**. Esses serviços podem utilizar **entidades de domínio puras** (ex: Dataclasses, Pydantic Models) para realizar os cálculos, garantindo que o núcleo do negócio seja testável isoladamente. O Model do Django é então tratado como um detalhe de implementação da camada de Infraestrutura, usado pelo serviço para persistir o resultado.
  4.  **Infraestrutura (Infrastructure):** Lida com detalhes técnicos. Implementa as interfaces definidas pela camada de Aplicação. Inclui o **Django ORM** (para acesso a dados), Celery (para filas), clientes de APIs externas, caches, etc. Os Models do Django, como implementação do padrão Active Record, atuam aqui como a ponte entre o domínio e a persistência.
- **Frontend (SPA):** O frontend será uma Single Page Application (SPA) desacoplada, comunicando-se com o backend exclusivamente via API RESTful.
- **Organização do Código-Fonte:** Adotaremos um **monorepo**, contendo tanto o código do backend (`backend/`) quanto do frontend (`frontend/`) no mesmo repositório Git.
  - **Justificativa:** Esta abordagem simplifica o gerenciamento de dependências, facilita a consistência entre a API e o cliente, e agiliza o pipeline de CI/CD, já que uma única alteração de contrato na API pode ser implementada e validada no frontend na mesma Pull Request.

---

## 2. Diagramas da Arquitetura (Modelo C4)

### 2.1. Nível 1: Diagrama de Contexto do Sistema (C1)

Este diagrama mostra o sistema IABANK no seu ambiente, interagindo com usuários e sistemas externos.

```mermaid
graph TD
    subgraph "Ecossistema IABANK"
        A[IABANK SaaS Platform]
    end

    U1[Gestor / Administrador] -->|Gerencia e supervisiona via Web App| A
    U2[Consultor / Cobrador] -->|Executa operações via Web App| A

    A -.->|Integração Futura| SE1[Bureaus de Crédito]
    A -.->|Integração Futura| SE2[Plataformas Bancárias (Pix, Open Finance)]
    A -.->|Integração Futura| SE3[Plataformas de Comunicação (WhatsApp)]

    style A fill:#1E90FF,stroke:#333,stroke-width:2px,color:#fff
```

### 2.2. Nível 2: Diagrama de Containers (C2)

Este diagrama detalha as principais unidades executáveis que compõem a plataforma IABANK.

```mermaid
graph TD
    U[Usuário] -->|HTTPS via Navegador| F[Frontend SPA <br><strong>[React, TypeScript, Vite]</strong>]

    subgraph "Infraestrutura na Cloud (Alta Disponibilidade)"
        F -->|API REST (JSON/HTTPS)| B[Backend API <br><strong>[Python, Django, DRF]</strong>]
        B -->|Leitura/Escrita (TCP/IP)| DB[(PostgreSQL Database <br><strong>[Cluster com Alta Disponibilidade (HA)]</strong>)]
        B -->|Enfileira tarefas| Q[Fila de Tarefas <br><strong>[Redis Cluster (HA)]</strong>]
        W[Worker <br><strong>[Celery]</strong>] -->|Consome de| Q
        W -->|"Usa código de serviço/domínio do"| B
        W -->|Leitura/Escrita| DB
    end

    style F fill:#61DAFB,stroke:#333,stroke-width:2px
    style B fill:#092E20,stroke:#333,stroke-width:2px,color:#fff
    style DB fill:#336791,stroke:#333,stroke-width:2px,color:#fff
    style Q fill:#DC382D,stroke:#333,stroke-width:2px,color:#fff
    style W fill:#3776AB,stroke:#333,stroke-width:2px,color:#fff
```

### 2.3. Nível 3: Diagrama de Componentes (C3) - Backend API

Este diagrama detalha os principais componentes modulares dentro do container "Backend API", mostrando um exemplo de fluxo para a criação de um empréstimo.

```mermaid
graph TD
    direction TB

    F[Frontend SPA] -->|1. POST /api/v1/loans| C1[Apresentação <br><strong>[operations.views.LoanViewSet]</strong>]

    subgraph "Container: Backend API (Django)"
        subgraph "App: operations"
            C1 --> |2. Chama| S_Loan[Aplicação: LoanService]
        end

        subgraph "App: customers"
            S_Loan --> |3. Usa| R_Customer[Infra: CustomerRepository]
        end

        subgraph "App: finance"
            S_Loan --> |5. Notifica| S_Finance[Aplicação: FinancialTransactionService]
        end

        subgraph "Camada de Domínio / Infraestrutura (Models)"
            R_Customer --> M_Customer[models.Customer]
            S_Loan -->|4. Persiste| M_Loan[models.Loan]
            S_Loan -->|4. Persiste| M_Installment[models.Installment]
            S_Finance --> |6. Persiste| M_Transaction[models.FinancialTransaction]
        end
    end

    M_Customer --> DB[(Database)]
    M_Loan --> DB
    M_Installment --> DB
    M_Transaction --> DB

    style S_Loan fill:#A2D4FF
    style S_Finance fill:#A2D4FF
    style R_Customer fill:#FFE6AA
```

---

### 2.A. Requisitos Regulatórios e de Negócio Essenciais

Esta seção define os requisitos não-funcionais e de negócio que são mandatórios para a operação de um sistema de gestão de empréstimos no Brasil. A arquitetura e os modelos de dados devem refletir e suportar estes pontos desde o início.

- **Conformidade Financeira (Banco Central do Brasil):**

  - **Cálculo do CET (Custo Efetivo Total):** A plataforma deve ser capaz de calcular e exibir o CET, tanto a taxa mensal quanto a anual, em todos os contratos e simulações. Este é um requisito legal.
  - **Cálculo do IOF (Imposto sobre Operações Financeiras):** O sistema deve calcular o IOF no momento da concessão do empréstimo, conforme as regras da Receita Federal. O valor do IOF impacta o valor liberado e o cálculo das parcelas.
  - **Lei da Usura:** As taxas de juros aplicadas devem respeitar os limites legais e ser configuráveis para se adaptar a mudanças na legislação.

- **Proteção ao Consumidor:**

  - **Período de Arrependimento:** O sistema deve suportar a lógica para o direito de arrependimento (cooling-off period), permitindo o cancelamento de um contrato sem ônus dentro do prazo legal (ex: 7 dias).

- **Lei Geral de Proteção de Dados (LGPD):**

  - **Definição de Papéis:** O IABANK atua como **Operador** dos dados em nome dos seus clientes (os tenants), que são os **Controladores**. A arquitetura multi-tenant e as permissões granulares devem garantir que cada Controlador tenha acesso apenas aos seus dados.
  - **Direitos dos Titulares:** A plataforma deve fornecer meios para que os Controladores possam atender às requisições dos titulares (ex: exportação, anonimização e exclusão de dados).

- **Integrações de Negócio Essenciais (Escopo V1):**
  - **Bureaus de Crédito (SPC/Serasa):** A integração para consulta de score de crédito é um requisito para a análise de risco na concessão de empréstimos.
  - **Gateways de Pagamento:** A integração com sistemas de pagamento (PIX, Boleto) é fundamental para a automação do recebimento das parcelas.

---

## 3. Descrição dos Componentes, Interfaces e Modelos de Domínio

### 3.1. Consistência dos Modelos de Dados (SSOT do Domínio - `models.py`)

Esta é a Fonte Única da Verdade para todas as entidades de dados do IABANK. Todos os modelos incluem um campo `tenant` para garantir isolamento de dados (multi-tenancy) desde o início.

**Tecnologia Chave:** `Django ORM`

```python
# iabank/core/models.py (Modelos base)
from django.db import models
from django.contrib.auth.models import AbstractUser

class Tenant(models.Model):
    name = models.CharField(max_length=255)
    # ... outros detalhes do tenant

class User(AbstractUser):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE, related_name="users")
    # ... outros campos de usuário

class BaseTenantModel(models.Model):
    """Modelo abstrato para garantir que todos os dados sejam vinculados a um tenant."""
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

# -------------------------------------------------------------

# iabank/customers/models.py
class Customer(BaseTenantModel):
    name = models.CharField(max_length=255)
    document_number = models.CharField(max_length=20) # CPF/CNPJ. Unicidade garantida por tenant.
    birth_date = models.DateField(null=True, blank=True)
    email = models.EmailField(null=True, blank=True)
    phone = models.CharField(max_length=20, null=True, blank=True)
    # ... outros campos de cliente

    class Meta:
        # Garante que o número do documento seja único por tenant.
        unique_together = ('tenant', 'document_number')

class Address(BaseTenantModel):
    """Modelo para armazenar endereços de clientes."""
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE, related_name='addresses')
    zip_code = models.CharField(max_length=10)
    street = models.CharField(max_length=255)
    number = models.CharField(max_length=20)
    complement = models.CharField(max_length=100, blank=True)
    neighborhood = models.CharField(max_length=100)
    city = models.CharField(max_length=100)
    state = models.CharField(max_length=2)
    is_primary = models.BooleanField(default=False) # Opcional: para marcar um endereço como principal

    class Meta:
        verbose_name_plural = "Addresses"

# iabank/operations/models.py
class Consultant(BaseTenantModel):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    balance = models.DecimalField(max_digits=10, decimal_places=2, default=0.00)
    # ... outros campos do consultor

class Loan(BaseTenantModel):
    class LoanStatus(models.TextChoices):
        IN_PROGRESS = 'IN_PROGRESS', 'Em Andamento'
        PAID_OFF = 'PAID_OFF', 'Finalizado'
        IN_COLLECTION = 'IN_COLLECTION', 'Em Cobrança'
        CANCELED = 'CANCELED', 'Cancelado'

    customer = models.ForeignKey('customers.Customer', on_delete=models.PROTECT, related_name='loans')
    consultant = models.ForeignKey(Consultant, on_delete=models.PROTECT, related_name='loans')
    principal_amount = models.DecimalField(max_digits=12, decimal_places=2) # Valor solicitado pelo cliente
    interest_rate = models.DecimalField(max_digits=5, decimal_places=2) # Taxa % mensal
    number_of_installments = models.PositiveSmallIntegerField()
    contract_date = models.DateField()
    first_installment_date = models.DateField()
    status = models.CharField(max_length=20, choices=LoanStatus.choices, default=LoanStatus.IN_PROGRESS)

    # --- Campos Regulatórios ---
    # Adicionados para conformidade com a legislação brasileira
    iof_amount = models.DecimalField(max_digits=10, decimal_places=2, help_text="Valor do IOF calculado na operação.")
    cet_annual_rate = models.DecimalField(max_digits=7, decimal_places=4, help_text="Custo Efetivo Total (taxa % anual).")
    cet_monthly_rate = models.DecimalField(max_digits=7, decimal_places=4, help_text="Custo Efetivo Total (taxa % mensal).")
    # ... outros campos do empréstimo

class Installment(BaseTenantModel):
    class InstallmentStatus(models.TextChoices):
        PENDING = 'PENDING', 'Pendente'
        PAID = 'PAID', 'Pago'
        OVERDUE = 'OVERDUE', 'Vencida'
        PARTIALLY_PAID = 'PARTIALLY_PAID', 'Parcialmente Pago'

    loan = models.ForeignKey(Loan, on_delete=models.CASCADE, related_name='installments')
    installment_number = models.PositiveSmallIntegerField()
    due_date = models.DateField()
    amount_due = models.DecimalField(max_digits=10, decimal_places=2)
    amount_paid = models.DecimalField(max_digits=10, decimal_places=2, default=0.00)
    payment_date = models.DateField(null=True, blank=True)
    status = models.CharField(max_length=20, choices=InstallmentStatus.choices, default=InstallmentStatus.PENDING)

# iabank/finance/models.py
class BankAccount(BaseTenantModel):
    name = models.CharField(max_length=100)
    agency = models.CharField(max_length=10, blank=True)
    account_number = models.CharField(max_length=20)
    initial_balance = models.DecimalField(max_digits=15, decimal_places=2, default=0.00)

class PaymentCategory(BaseTenantModel):
    name = models.CharField(max_length=100)

class CostCenter(BaseTenantModel):
    name = models.CharField(max_length=100)

class Supplier(BaseTenantModel):
    name = models.CharField(max_length=255)
    document_number = models.CharField(max_length=20, null=True, blank=True)

class FinancialTransaction(BaseTenantModel):
    class TransactionType(models.TextChoices):
        INCOME = 'INCOME', 'Receita'
        EXPENSE = 'EXPENSE', 'Despesa'

    description = models.CharField(max_length=255)
    amount = models.DecimalField(max_digits=12, decimal_places=2)
    transaction_date = models.DateField()
    is_paid = models.BooleanField(default=False)
    payment_date = models.DateField(null=True, blank=True)
    type = models.CharField(max_length=10, choices=TransactionType.choices)

    bank_account = models.ForeignKey(BankAccount, on_delete=models.PROTECT)
    category = models.ForeignKey(PaymentCategory, on_delete=models.SET_NULL, null=True, blank=True)
    cost_center = models.ForeignKey(CostCenter, on_delete=models.SET_NULL, null=True, blank=True)
    supplier = models.ForeignKey(Supplier, on_delete=models.SET_NULL, null=True, blank=True)
    installment = models.ForeignKey('operations.Installment', on_delete=models.SET_NULL, null=True, blank=True, related_name='payments')
```

### 3.1.1. Detalhamento dos DTOs e Casos de Uso

**Tecnologia Chave:** `Pydantic`

```python
# iabank/operations/dtos.py
from pydantic import BaseModel, Field
from datetime import date
from decimal import Decimal

class LoanCreateDTO(BaseModel):
    customer_id: int
    consultant_id: int
    principal_amount: Decimal = Field(..., gt=0)
    interest_rate: Decimal = Field(..., ge=0)
    number_of_installments: int = Field(..., gt=0)
    contract_date: date
    first_installment_date: date

class LoanListDTO(BaseModel):
    id: int
    customer_name: str
    principal_amount: Decimal
    status: str
    contract_date: date

    class Config:
        orm_mode = True

class CustomerCreateDTO(BaseModel):
    name: str
    document_number: str
    email: str | None = None
    phone: str | None = None
    # ... outros campos para criação
```

### 3.2. Camada de Apresentação (UI)

#### Decomposição em Telas/Componentes

- **Telas (Views):** `Dashboard`, `LoanList`, `LoanDetails`, `NewLoanWizard`, `CustomerList`, `CustomerForm`, `AccountsPayable`, `Login`.
- **Componentes Reutilizáveis:** `SmartTable` (com filtros, paginação, seleção de colunas), `KPI_Card`, `WizardStepper`, `StatusBadge`, `FormInput`, `DatePicker`.

#### Contrato de Dados da View (ViewModel)

**Tecnologia Chave:** `TypeScript Interfaces`

```typescript
// frontend/src/features/loans/types/index.ts

// ViewModel para a tela de listagem de empréstimos
export interface LoanListViewModel {
  id: number;
  customerName: string;
  principalAmountFormatted: string; // "R$ 10.000,00"
  status: "IN_PROGRESS" | "PAID_OFF" | "IN_COLLECTION" | "CANCELED";
  statusLabel: string; // "Em Andamento"
  statusColor: "yellow" | "green" | "red" | "gray"; // Para o StatusBadge
  contractDateFormatted: string; // "dd/mm/yyyy"
  installmentsProgress: string; // "3/12"
}

// Mapeamento: Derivado do LoanListDTO do backend.
// O Serializer no DRF irá pré-calcular campos como `customer_name`.
// O frontend irá formatar valores monetários e datas e mapear o status para label/cor.

// frontend/src/features/customers/types/index.ts

// ViewModel para a tela de listagem de clientes
export interface CustomerListViewModel {
  id: number;
  name: string;
  documentNumberFormatted: string; // "123.456.789-00"
  phone: string;
  city: string;
  activeLoansCount: number;
}
```

---

## 4. Descrição Detalhada da Arquitetura Frontend

A arquitetura do frontend seguirá formalmente a metodologia **[Feature-Sliced Design (FSD)](https://feature-sliced.design/)**, um padrão arquitetural para aplicações frontend que promove alta coesão, baixo acoplamento e escalabilidade.

- **Padrão Arquitetural:** O código é organizado por fatias verticais de negócio (`features`), em vez de fatias horizontais por tipo de arquivo. Isso significa que toda a lógica de uma funcionalidade (ex: gestão de empréstimos) — UI, API, estado e modelo — reside em um único diretório, facilitando a navegação, manutenção e o trabalho em paralelo.

- **Estrutura de Diretórios Proposta (`frontend/src/`):**

  ```
  src/
  ├── app/                # Configuração global da aplicação (providers, store, router, styles)
  ├── pages/              # Componentes de página, que compõem layouts a partir das features
  ├── features/           # Funcionalidades de negócio (ex: loan-list, customer-form)
  │   ├── loan-list/
  │   │   ├── api/        # Hooks de API (TanStack Query) e definições de requisição
  │   │   ├── components/ # Componentes específicos desta feature (ex: LoanFilterPanel)
  │   │   ├── model/      # Types, schemas de validação (Zod)
  │   │   └── ui/         # O componente principal da feature (ex: LoanListTable.tsx)
  │   └── ...
  ├── entities/           # Componentes e lógica de entidades de negócio (ex: LoanCard, CustomerAvatar)
  └── shared/             # Código reutilizável e agnóstico de negócio
      ├── api/            # Configuração do cliente Axios/Fetch global
      ├── config/         # Constantes, configurações de ambiente
      ├── lib/            # Funções utilitárias, helpers, hooks genéricos
      └── ui/             # Biblioteca de componentes de UI puros (Button, Input, Table)
  ```

- **Estratégia de Gerenciamento de Estado:**

  - **Estado do Servidor:** **TanStack Query (React Query)** será a fonte da verdade para todos os dados assíncronos vindos da API. Ele gerenciará caching, revalidação, mutações e estados de loading/error de forma declarativa.
  - **Estado Global do Cliente:** **Zustand** será usado para estados globais síncronos e de baixa frequência de atualização, como informações do usuário autenticado, tema da UI ou estado de um menu lateral.
  - **Estado Local do Componente:** Os hooks nativos do React (`useState`, `useReducer`) serão usados para estado efêmero e contido dentro de um único componente (ex: estado de um input de formulário).

- **Fluxo de Dados:**

  1.  O usuário interage com um componente na camada `pages` ou `features`.
  2.  O componente dispara um hook da camada `features/api` (ex: `useLoans()`).
  3.  TanStack Query faz a chamada à API backend através do cliente configurado em `shared/api`.
  4.  A resposta da API é armazenada no cache do TanStack Query.
  5.  O hook `useLoans()` retorna os dados, que são passados para os componentes `entities` ou `shared/ui` para renderização.
  6.  Mutações (Create, Update, Delete) usam o hook `useMutation` do TanStack Query, que automaticamente revalida os dados relevantes após o sucesso da operação.

- **Sincronização de Tipos (Backend-Frontend):**
  Para eliminar a dessincronização de contratos entre a API e o cliente, adotaremos uma abordagem _schema-first_ para os tipos.
  1.  O Django REST Framework será configurado para gerar um schema **OpenAPI 3** (`openapi.json`) que descreve todos os endpoints, DTOs e modelos.
  2.  Este arquivo de schema será versionado no repositório.
  3.  No frontend, utilizaremos a ferramenta **`openapi-typescript-codegen`** para gerar automaticamente as interfaces TypeScript e os clientes de API a partir do arquivo `openapi.json`.
  4.  Um script (`pnpm gen:api-types`) será adicionado ao `package.json` do frontend, e o pipeline de CI irá verificar se os tipos gerados estão atualizados com o schema, garantindo consistência total.

---

## 5. Definição das Interfaces Principais

Exemplo de interface para um serviço de aplicação no backend.

**Tecnologia Chave:** Python (Lógica Pura), Pydantic

```python
# iabank/operations/services.py
from .dtos import LoanCreateDTO
from .models import Loan
from ..customers.models import Customer
from .repositories import LoanRepository # Abstração do acesso a dados

class LoanService:
    def __init__(self, loan_repo: LoanRepository):
        """
        O serviço é inicializado com suas dependências (Injeção de Dependência).
        A configuração (ex: taxas padrão) pode ser passada aqui ou obtida
        de um serviço de configuração.
        """
        self.loan_repo = loan_repo

    def create_loan(self, tenant_id: int, loan_data: LoanCreateDTO) -> Loan:
        """
        Cria um novo empréstimo e suas parcelas.
        Responsabilidade: Orquestrar a lógica de negócio, validações e persistência
        dentro de uma transação atômica.
        """
        # 1. Validações de negócio (ex: cliente existe, consultor ativo)
        # 2. Lógica de cálculo de parcelas
        # 3. Persistência usando o repositório
        # 4. Criação de transações financeiras associadas
        # ...
        pass
```

---

## 6. Gerenciamento de Dados

- **Persistência e Acesso:** O **ORM do Django** será a principal ferramenta de acesso a dados. O padrão Repository pode ser usado em módulos complexos para abstrair as queries do ORM, facilitando os testes.
- **Gerenciamento de Schema:** As migrações de banco de dados serão gerenciadas pelo sistema nativo do Django (`makemigrations`, `migrate`), garantindo a evolução consistente do schema.
- **Seed de Dados:** Para ambientes de desenvolvimento e teste, serão criados **comandos de gerenciamento customizados** do Django (ex: `python manage.py seed_data`). Estes comandos utilizarão a biblioteca **`factory-boy`** para gerar dados fictícios, porém realistas e consistentes (respeitando as relações de tenant).

### 6.1. Backup e Restore

A estratégia de backup e restauração do banco de dados PostgreSQL é projetada para minimizar a perda de dados (RPO baixo) e o tempo de inatividade (RTO baixo).

- **Objetivos:**

  - **Recovery Point Objective (RPO):** < 5 minutos.
  - **Recovery Time Objective (RTO):** < 1 hora.

- **Estratégia Técnica:**
  - **Point-in-Time Recovery (PITR):** Será implementado o arquivamento contínuo do Write-Ahead Log (WAL). Isso nos permite restaurar o banco de dados para qualquer ponto no tempo, não apenas para o momento do último backup completo. Os arquivos WAL serão enviados para um storage de objetos (ex: AWS S3) em tempo quase real.
  - **Backups Completos:** Um backup completo da base de dados será realizado diariamente durante a janela de menor utilização.
  - **Retenção:**
    - Arquivos WAL: Retidos por 14 dias.
    - Backups Diários: Retidos por 30 dias.
    - Backups Mensais: Retidos por 1 ano para fins de conformidade.
  - **Testes de Restauração:** O processo de restauração será testado trimestralmente em um ambiente isolado para garantir a integridade dos backups e a eficácia do procedimento.

### 6.2. Trilha de Auditoria (Audit Trail)

Para garantir a rastreabilidade e a conformidade, um sistema de trilha de auditoria será implementado para registrar todas as alterações em modelos de dados críticos.

- **Tecnologia Chave:** `django-simple-history`. Esta biblioteca foi escolhida por sua capacidade de armazenar um histórico completo de cada registro, incluindo o usuário que fez a alteração e o tipo de alteração (Criação, Atualização, Exclusão). Ela efetivamente versiona os modelos.

- **Modelos a Serem Auditados (Escopo Inicial):**

  - `Loan`
  - `Installment`
  - `Customer`
  - `FinancialTransaction`
  - `User`

- **Casos de Uso:**
  - **Análise Forense:** Investigar alterações de dados não autorizadas ou incorretas.
  - **Suporte ao Cliente:** Entender o histórico de um empréstimo ou cliente para resolver disputas.
  - **Reversão de Dados (Rollback):** A biblioteca permite reverter um registro para um estado anterior, o que pode ser uma ferramenta poderosa para correções de emergência.

### 6.3. Estratégia de Indexação Multi-Tenant

Para garantir a performance das consultas em um ambiente com banco de dados compartilhado, é crucial que a estratégia de indexação seja consciente do isolamento por tenant.

- **Princípio Chave:** A coluna `tenant_id` **deve ser a primeira coluna** na maioria dos índices compostos. Isso permite que o planejador de consultas do PostgreSQL filtre eficientemente o conjunto de dados para um único tenant antes de aplicar os filtros subsequentes, reduzindo drasticamente o escopo da busca.

- **Exemplo de Implementação (Django `models.Index`):**

  ```python
  # iabank/operations/models.py

  class Loan(BaseTenantModel):
    # ... campos do modelo

    class Meta:
      indexes = [
        # Índice otimizado para buscar empréstimos de um cliente específico dentro de um tenant
        models.Index(fields=['tenant', 'customer']),
        # Índice otimizado para buscar empréstimos por status dentro de um tenant
        models.Index(fields=['tenant', 'status']),
      ]
  ```

---

## 7. Estrutura de Diretórios Proposta (Monorepo)

```
iabank/
├── .github/
│   └── workflows/
│       └── main.yml           # Pipeline de CI/CD
├── backend/
│   ├── src/
│   │   └── iabank/
│   │       ├── __init__.py
│   │       ├── asgi.py
│   │       ├── settings.py
│   │       ├── urls.py
│   │       ├── wsgi.py
│   │       ├── core/            # App com modelos base, middlewares, etc.
│   │       ├── customers/
│   │       │   ├── __init__.py
│   │       │   ├── models.py
│   │       │   ├── admin.py
│   │       │   ├── apps.py
│   │       │   ├── serializers.py
│   │       │   ├── views.py
│   │       │   └── tests/
│   │       │       ├── __init__.py
│   │       │       ├── test_models.py
│   │       │       └── test_views.py
│   │       ├── finance/         # App Financeiro
│   │       ├── operations/      # App Operacional (Empréstimos, Consultores)
│   │       └── users/           # App de Usuários e Permissões
│   ├── manage.py
│   ├── Dockerfile
│   └── pyproject.toml
├── frontend/
│   ├── public/
│   ├── src/                 # Estrutura detalhada na seção 4
│   ├── Dockerfile
│   ├── package.json
│   ├── tsconfig.json
│   └── vite.config.ts
├── tests/
│   └── integration/
│       ├── __init__.py
│       └── test_full_loan_workflow.py
├── docker-compose.yml
├── .gitignore
├── .pre-commit-config.yaml
├── CHANGELOG.md
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

---

## 8. Arquivo `.gitignore` Proposto

```
# Byte-compiled / optimized / DLL files
__pycache__/
*.py[cod]
*$py.class

# C extensions
*.so

# Distribution / packaging
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
share/python-wheels/
*.egg-info/
.installed.cfg
*.egg
MANIFEST

# PyInstaller
#  Usually these files are written by a python script from a template
#  before PyInstaller builds the exe, so as to inject date/other infos into it.
*.manifest
*.spec

# Installer logs
pip-log.txt
pip-delete-this-directory.txt

# Unit test / coverage reports
htmlcov/
.tox/
.nox/
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
*.py,cover
.hypothesis/
.pytest_cache/
cover/

# Translations
*.mo
*.pot

# Django stuff:
*.log
local_settings.py
db.sqlite3
db.sqlite3-journal

# Environments
.env
.venv
env/
venv/
ENV/
env.bak/
venv.bak/

# IDE files
.idea/
.vscode/
*.swp
*.swo

# node.js
node_modules/
dist/
dist-ssr/
*.local
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# Docker
docker-compose.override.yml

# OS files
.DS_Store
Thumbs.db
```

---

## 9. Arquivo `README.md` Proposto

````markdown
# IABANK

[![Status](https://img.shields.io/badge/status-em_desenvolvimento-yellow)](https://github.com/your-org/iabank)
[![CI/CD](https://github.com/your-org/iabank/actions/workflows/main.yml/badge.svg)](https://github.com/your-org/iabank/actions)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Sistema de gestão de empréstimos moderno e eficiente.

## Sobre o Projeto

O IABANK é uma plataforma Web SaaS robusta e segura, desenvolvida em Python e React, projetada para a gestão completa de empréstimos (end-to-end). A arquitetura é multi-tenant e foi concebida para ser escalável, intuitiva e adaptável.

## Stack Tecnológica

- **Backend:** Python 3.10+, Django, Django REST Framework
- **Frontend:** React 18+, TypeScript, Vite, Tailwind CSS
- **Banco de Dados:** PostgreSQL
- **Cache & Fila de Tarefas:** Redis, Celery
- **Containerização:** Docker, Docker Compose

## Como Começar

### Pré-requisitos

- Docker e Docker Compose
- Node.js e pnpm (para o frontend)
- Python e Poetry (para o backend)

### Instalação e Execução

1.  Clone o repositório:

    ```bash
    git clone https://github.com/your-org/iabank.git
    cd iabank
    ```

2.  Crie um arquivo `.env` na raiz do projeto a partir do `.env.example`.

3.  Suba os contêineres Docker:

    ```bash
    docker-compose up -d --build
    ```

4.  A aplicação estará disponível em:
    - Frontend: `http://localhost:5173`
    - Backend API: `http://localhost:8000/api/`

## Como Executar os Testes

Para executar os testes do backend, entre no contêiner do Django e use o `pytest`:

```bash
docker-compose exec backend bash
pytest
```

## Estrutura do Projeto

O projeto é um monorepo com duas pastas principais:

- `/backend`: Contém a aplicação Django (API).
- `/frontend`: Contém a Single Page Application (SPA) em React.

Consulte o [Blueprint Arquitetural](docs/architecture.md) para mais detalhes.
````

---

## 10. Arquivo `LICENSE` Proposto

```markdown
MIT License

Copyright (c) [Ano] [Nome do Proprietário do Copyright]

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## 11. Arquivo `CONTRIBUTING.md` Proposto

````markdown
# Como Contribuir para o IABANK

Agradecemos o seu interesse em contribuir! Para garantir a qualidade e a consistência do projeto, pedimos que siga estas diretrizes.

## Processo de Desenvolvimento

1.  **Siga o Blueprint:** Todas as contribuições devem estar alinhadas com o [Blueprint Arquitetural](docs/architecture.md). Mudanças na arquitetura devem ser discutidas e aprovadas antes da implementação.
2.  **Crie uma Issue:** Antes de começar a trabalhar, abra uma issue descrevendo o bug ou a feature.
3.  **Crie um Branch:** Crie um branch a partir do `main` com um nome descritivo (ex: `feature/add-loan-export` ou `fix/login-bug`).
4.  **Desenvolva com Testes:** Todo novo código de negócio deve ser acompanhado de testes unitários e/ou de integração.
5.  **Abra um Pull Request:** Ao concluir, abra um Pull Request para o branch `main`. Descreva as suas alterações e vincule a issue correspondente.

## Qualidade de Código

Utilizamos ferramentas para garantir um padrão de código consistente.

- **Backend (Python):**
  - **Formatador:** [Black](https://github.com/psf/black)
  - **Linter:** [Ruff](https://github.com/astral-sh/ruff)
- **Frontend (TypeScript):**
  - **Formatador:** [Prettier](https://prettier.io/)
  - **Linter:** [ESLint](https://eslint.org/)

### Diretrizes para a Camada de Domínio

Para manter a separação de responsabilidades entre as camadas, siga estas diretrizes ao implementar a lógica de negócio:

- **Lógica nos Models e Managers do Django:**

  - **O que colocar aqui:** Regras de negócio que operam sobre os dados de **um único registro** e não têm dependências externas (como outros serviços ou APIs).
      - **Exemplos:** - Um método `@property` para verificar o estado de um objeto: `loan.is_overdue`. - Uma validação de campo no método `clean()`. - Um método de Manager para encapsular uma query complexa e reutilizável: `Loan.objects.get_active_loans_for_customer(customer_id)`.

- **Lógica nos Serviços da Camada de Aplicação:**
  - **O que colocar aqui:** Orquestrações complexas que envolvem **múltiplos modelos**, cálculos que necessitam de dados de fontes diferentes, ou qualquer ação que interaja com a camada de infraestrutura (enviar e-mails, enfileirar tarefas, chamar APIs externas).
  - **Exemplos:**
    - `LoanService.create_loan()`: Envolve a criação de um `Loan`, múltiplas `Installment`, e uma `FinancialTransaction`, tudo dentro de uma transação atômica.
    - `PaymentService.process_installment_payment()`: Envolve atualizar uma `Installment`, possivelmente o `Loan`, criar uma `FinancialTransaction` e notificar o cliente.

### Ganchos de Pre-commit

Configuramos ganchos de pre-commit usando a ferramenta `pre-commit` para executar essas checagens automaticamente antes de cada commit. Para instalar, execute:

```bash
pip install pre-commit
pre-commit install
```

Qualquer código que não passe nas verificações do linter/formatador será bloqueado.

## Padrão de Documentação

- **Python:** Todas as funções públicas, classes e métodos devem ter docstrings no estilo [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings).
- **React:** Componentes complexos devem ter comentários explicando seu propósito, props e estado.
````

---

## 12. Estrutura do `CHANGELOG.md`

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Initial project structure and architecture blueprint.

## [0.1.0] - YYYY-MM-DD

### Added

- ...
```

---

## 13. Estratégia de Configuração e Ambientes

A configuração será gerenciada usando a biblioteca **`django-environ`**.

- Um arquivo `.env` na raiz do `backend/` conterá as variáveis de ambiente para desenvolvimento local (ex: `DATABASE_URL`, `SECRET_KEY`, `DEBUG=True`). Este arquivo não deve ser versionado.
- Um arquivo `.env.example` será versionado para servir como template.
- Em ambientes de homologação e produção, as configurações serão injetadas diretamente como **variáveis de ambiente** no provedor de nuvem ou no orquestrador de contêineres. Isso garante que segredos nunca sejam armazenados em código.
- O arquivo `settings.py` do Django será o único ponto de leitura dessas variáveis, usando `env('VARIABLE_NAME', default='value')`.

---

## 14. Estratégia de Observabilidade Completa

- **Logging Estruturado:** Utilizaremos a biblioteca **`structlog`** para gerar logs em formato **JSON**. Em desenvolvimento, os logs serão formatados para leitura humana no console. Em produção, o output JSON será facilmente ingerido por serviços como Datadog, Sentry ou um stack ELK. Os logs incluirão contexto automático, como `request_id`, `user_id` e `tenant_id`. Este contexto será injetado automaticamente através de um **Middleware** customizado do Django, que associará essas informações a cada requisição e as disponibilizará para o `structlog`.
- **Métricas de Negócio:** Serão expostas via um endpoint `/metrics` (usando `django-prometheus`). Métricas chave incluem: `loans_created_total`, `installments_paid_total`, `overdue_amount_total`, `api_request_latency_seconds`. Dashboards no Grafana ou similar visualizarão essas métricas.
- **Distributed Tracing:** Embora seja um monolito, o uso de `OpenTelemetry` será considerado desde o início para rastrear requisições. Isso será crucial quando futuras integrações (agentes de IA, sistemas externos) forem adicionadas.
- **Health Checks e SLIs/SLOs:**
  - Um endpoint `/health` será criado, retornando `200 OK` se os serviços essenciais (conexão com DB e Redis) estiverem funcionando.
  - **SLI:** Latência da API do endpoint `GET /api/v1/loans/` < 500ms.
  - **SLO:** 99.9% de disponibilidade mensal do sistema.
- **Alerting Inteligente:** **Sentry** será integrado para captura automática de exceções no backend e frontend. Alertas serão configurados para picos de erros (`5xx`), aumento de latência e **anomalias nas métricas de negócio (ex: aumento súbito da taxa de inadimplência, queda no volume de pagamentos)**, com integração ao Slack/PagerDuty.

---

## 15. Estratégia de Testes Detalhada

- **Estrutura e Nomenclatura:**

  - **Testes Unitários:** Residem dentro de cada app Django, em `<app_name>/tests/`. Validam a lógica de um único componente isoladamente (ex: um método de modelo, uma função de serviço com dependências mockadas).
  - **Testes de Integração:** Residem no diretório de alto nível `tests/integration/`. Validam a interação entre múltiplos componentes (ex: um fluxo completo de API, desde a view até o banco de dados).
  - **Convenção de Nomenclatura:** Todos os arquivos de teste seguirão o padrão `test_<nome_do_modulo>.py` (ex: `test_models.py`, `test_serializers.py`).

- **Ferramentas:**

  - **Executor:** `pytest`
  - **Cliente de API:** Cliente de teste do Django REST Framework (`APIClient`).
  - **Dados Fictícios:** `factory-boy`.
  - **Mocks:** `unittest.mock`.

- **Padrões de Teste de Integração:**

  - **Uso de Factories:** `factory-boy` será mandatório para criar o estado inicial do banco de dados para os testes, garantindo consistência e legibilidade.
  - **Simulação de Autenticação:** Usaremos `api_client.force_authenticate(user=self.user)` para simular um usuário logado em testes de API, evitando o fluxo completo de login.

- **Padrões Obrigatórios para Test Data Factories (`factory-boy`):**

  - **Princípio da Herança Explícita de Contexto (Multi-tenancy):**

    - **Regra:** Factories que criam objetos aninhados **DEVEM** propagar o `tenant` da factory pai para todas as sub-factories.
    - **Exemplo Mandatório:**

      ```python
      # iabank/operations/tests/factories.py
      import factory
      from iabank.core.tests.factories import TenantFactory
      from iabank.customers.tests.factories import CustomerFactory
      from ..models import Loan

      class LoanFactory(factory.django.DjangoModelFactory):
          class Meta:
              model = Loan

          tenant = factory.SubFactory(TenantFactory)
          # CRÍTICO: Propagar o tenant para o cliente relacionado
          customer = factory.SubFactory(CustomerFactory, tenant=factory.SelfAttribute('..tenant'))
          # ... outros campos
      ```

  - **Testes Obrigatórios para Factories (Meta-testes):**

    - **Regra:** Para cada `factories.py`, um arquivo `test_factories.py` deve existir para validar a consistência dos dados gerados.
    - **Exemplo de teste crítico:**

      ```python
      # iabank/operations/tests/test_factories.py
      from django.test import TestCase
      from .factories import LoanFactory, TenantFactory

      class OperationFactoriesTestCase(TestCase):
          def test_loan_factory_tenant_consistency(self):
              """Garante que LoanFactory propaga o tenant para o cliente."""
              tenant = TenantFactory()
              loan = LoanFactory(tenant=tenant)

              self.assertEqual(loan.tenant, tenant)
              self.assertEqual(loan.customer.tenant, tenant, "O tenant do cliente deve ser o mesmo do empréstimo.")
      ```

### 15.1. Testes End-to-End (E2E)

- **Objetivo:** Validar fluxos de negócio críticos da perspectiva do usuário final, simulando interações reais no navegador. Estes testes garantem que a integração entre o frontend e o backend está funcionando corretamente.
- **Ferramentas:** **Cypress** ou **Playwright**.
- **Escopo:** Os testes E2E focarão nos "caminhos felizes" das jornadas mais importantes, como:
  1. Criação de um novo empréstimo, desde o login até a confirmação.
  2. Registro de um pagamento de parcela e verificação da atualização do status.
  3. Geração de um relatório financeiro.
- **Localização:** Os testes residirão em um diretório `e2e/` na raiz do monorepo e serão executados em um estágio separado do pipeline de CI, contra um ambiente de homologação (`staging`).

---

## 16. Estratégia de CI/CD (Integração e Implantação Contínuas)

- **Ferramenta:** GitHub Actions (arquivo de workflow em `.github/workflows/main.yml`).
- **Gatilhos:** Em cada `push` para o branch `main` e em cada abertura/atualização de `Pull Request`.
- **Estágios do Pipeline:**
  1. **CI (Validação em PR):**
     - **Setup:** Checkout do código, setup do Python e Node.js, instalação de dependências (com cache).
     - **Lint & Format:** Executa `ruff`, `black`, `eslint`, `prettier` para garantir a qualidade do código.
     - **Test:** Executa a suíte completa de testes unitários e de integração do backend e do frontend.
     - **Build:** Executa o build de produção do frontend (`pnpm build`) para garantir que não há erros.
  2. **CD (Deploy em `main`):**
     - (Após sucesso da etapa de CI)
     - **Build & Push Images:** Constrói as imagens Docker do backend e frontend e as envia para um registro (ex: Docker Hub, AWS ECR). As imagens são tagueadas com o hash do commit.
     - **Deploy:** Um job separado (usando environments do GitHub Actions) se conecta ao ambiente de produção (ex: AWS, Heroku) e atualiza os serviços para usar as novas imagens.
- **Rollback e Migrações de Banco de Dados:**
  - **Estratégia de Rollback da Aplicação (Reativa):** A abordagem inicial será reverter o commit problemático no branch `main`, permitindo que o pipeline de CD implante a versão estável anterior.
    - **Evolução (Proativa):** Conforme a maturidade do produto aumentar, evoluiremos para estratégias de implantação mais robustas, como **Blue-Green Deployment** ou **Canary Releases**. Essas estratégias minimizam o tempo de inatividade e permitem rollbacks quase instantâneos.
    - **Estratégia de Migração Segura:** Para evitar situações em que um rollback de aplicação é bloqueado por uma migração de banco de dados destrutiva, seguiremos o princípio de **migrações retrocompatíveis**. Alterações complexas (ex: renomear uma coluna) serão divididas em múltiplos deploys (ex: 1. Adicionar nova coluna; 2. Deploy da aplicação para escrever em ambas; 3. Backfill de dados; 4. Deploy da aplicação para ler apenas da nova coluna; 5. Remover coluna antiga). Isso garante que a aplicação possa ser revertida para a versão anterior a qualquer momento sem corromper dados.
- **Otimização do Pipeline (Path Filtering):**
  - Para manter a agilidade em um monorepo, o pipeline será configurado para executar jobs de forma seletiva. Usando o `paths` filter do GitHub Actions, uma alteração apenas no diretório `/frontend` irá disparar os testes e o build do frontend, mas não a suíte de testes completa do backend, e vice-versa. Isso reduz drasticamente o tempo de feedback para os desenvolvedores e o custo computacional.

---

## 17. Estratégia de Versionamento da API

A API será versionada via **URL**. Todas as rotas serão prefixadas com a versão, ex: `/api/v1/loans/`. Isso permite introduzir uma `v2` no futuro sem quebrar o cliente `v1` existente. A versão será gerenciada pelo `URLconf` do Django e pelo `DefaultRouter` do DRF.

---

## 18. Padrão de Resposta da API e Tratamento de Erros

- **Resposta de Sucesso (`2xx`):**
  ```json
  {
    "data": {
      /* objeto ou lista de objetos */
    },
    "meta": {
      "pagination": {
        "count": 100,
        "page": 1,
        "pages": 10,
        "per_page": 10
      }
    }
  }
  ```
- **Resposta de Erro (`4xx`, `5xx`):**
  ```json
  {
    "errors": [
      {
        "status": "422",
        "code": "validation_error",
        "source": { "pointer": "/data/attributes/principal_amount" },
        "detail": "Este campo não pode ser menor ou igual a zero."
      }
    ]
  }
  ```
  Um `ExceptionHandler` customizado no DRF será implementado para capturar todas as exceções e formatá-las neste padrão consistente.

---

## 19. Estratégia de Segurança Abrangente

- **Threat Modeling Básico:**
  - **Ameaça:** Acesso não autorizado a dados de outro tenant.
    - **Mitigação:** Middleware de Tenant obrigatório que injeta `tenant_id` em todas as queries. Testes de integração que verificam falha ao tentar acessar recursos de outro tenant.
  - **Ameaça:** Injeção de SQL.
    - **Mitigação:** Uso exclusivo do ORM do Django, que parametriza todas as queries.
  - **Ameaça:** Vazamento de segredos no código.
    - **Mitigação:** Uso de `pre-commit` hooks para detectar segredos e política rigorosa de uso de variáveis de ambiente.
- **Secrets Management:** Em produção, utilizaremos um serviço dedicado como **AWS Secrets Manager** ou **HashiCorp Vault**. As aplicações receberão permissões IAM para acessar apenas os segredos de que necessitam em tempo de execução.
- **Compliance (LGPD):** A arquitetura suporta os princípios da LGPD:
  - **RBAC:** O módulo "Usuários e Permissões" implementa o controle de acesso granular.
  - **Isolamento:** A arquitetura multi-tenant garante o isolamento dos dados dos titulares.
  - **Auditoria:** O módulo "Logs de Atividade" registra todas as operações críticas.
- **Security by Design:**
  - **Autenticação:** A autenticação será baseada em JWT (JSON Web Tokens), seguindo as melhores práticas de segurança:
    - **`access_token`:** Token de curta duração (ex: 15 minutos), assinado, contendo as permissões do usuário (`payload`). Será armazenado em memória no estado do frontend (ex: Zustand), sendo enviado no header `Authorization` de cada requisição.
    - **`refresh_token`:** Token de longa duração (ex: 7 dias), opaco, usado para obter novos `access_tokens`. Será armazenado em um cookie **`HttpOnly`**, **`Secure`** e **`SameSite=Strict`**. Esta configuração impede que o token seja acessado por JavaScript no navegador, mitigando o risco de roubo via ataques XSS.
    - **Autorização:** Verificação de permissões em nível de API View, usando as classes de permissão do DRF.
    - **Defesa em Profundidade:** HTTPS obrigatório, headers de segurança (HSTS, CSP), proteção contra CSRF e XSS (nativa no Django/React).
- **Criptografia de Dados:**
  - **Criptografia em Repouso (At Rest):** Todos os dados armazenados no banco de dados e em seus backups serão criptografados em repouso. Esta funcionalidade será habilitada no nível do provedor de nuvem (ex: usando AWS KMS com RDS).
  - **Criptografia em Nível de Campo:** Dados PII (Informações Pessoais Identificáveis) sensíveis, como `document_number`, serão avaliados para criptografia em nível de campo utilizando bibliotecas como `django-cryptography`, adicionando uma camada extra de proteção.
- **Autenticação Multifator (MFA):** A MFA será um recurso mandatório para todos os usuários com perfis administrativos ou que tenham permissão para realizar operações financeiras críticas. A implementação suportará aplicativos autenticadores baseados em TOTP (ex: Google Authenticator).

---

## 20. Justificativas e Trade-offs

- **Monolito vs. Microserviços:**
  - **Decisão:** Monolito Modular.
  - **Justificativa:** Para a fase inicial e o tamanho da equipe, a complexidade operacional de microserviços (deploy, monitoramento, comunicação inter-serviços) superaria os benefícios. O monolito permite um desenvolvimento mais rápido e transações ACID mais simples. A estrutura modular interna mantém o código organizado e preparado para uma futura extração, se o sistema crescer em complexidade e escala.
  - **Trade-off:** Menor flexibilidade para escalar componentes individualmente e risco de acoplamento acidental se a disciplina de modularidade não for mantida.

---

## 21. Exemplo de Bootstrapping/Inicialização

A inicialização e injeção de dependências serão gerenciadas implicitamente pelo Django. Para os serviços da camada de aplicação, a instanciação ocorrerá dentro das Views do DRF, onde as dependências (como repositórios) serão passadas:

```python
# iabank/operations/views.py
from rest_framework.viewsets import ModelViewSet
from .services import LoanService
from .repositories import DjangoLoanRepository

class LoanViewSet(ModelViewSet):
    # ... queryset, serializer_class, etc.

    def get_loan_service(self):
        # A injeção de dependência acontece aqui
        loan_repo = DjangoLoanRepository()
        return LoanService(loan_repo=loan_repo)

    def perform_create(self, serializer):
        loan_service = self.get_loan_service()
        tenant_id = self.request.user.tenant_id
        # O DTO é construído a partir dos dados validados pelo serializer
        loan_dto = LoanCreateDTO(**serializer.validated_data)
        loan_service.create_loan(tenant_id=tenant_id, loan_data=loan_dto)
```

### Nota sobre Inversão de Dependência

A abordagem acima, onde a `ViewSet` instancia diretamente o `DjangoLoanRepository`, é uma forma pragmática e simples de injeção de dependência manual. Ela funciona bem e já desacopla o serviço da View.

**Evolução Futura:** Para um desacoplamento ainda mais rigoroso, alinhado com o Princípio da Inversão de Dependência (D do SOLID), poderíamos adotar uma biblioteca de Injeção de Dependência (DI) como `python-dependency-injector`. Isso permitiria centralizar a configuração de qual implementação concreta (ex: `DjangoLoanRepository`) deve ser usada para uma interface (ex: `LoanRepository`), tornando a `ViewSet` completamente agnóstica à implementação da persistência. Esta é uma evolução a ser considerada se a complexidade das dependências aumentar significativamente.

---

## 22. Estratégia de Evolução do Blueprint

- **Versionamento:** O próprio Blueprint será versionado usando **Semantic Versioning**. Mudanças que não quebram a estrutura (adição de um novo módulo) são `MINOR`. Mudanças que alteram a arquitetura fundamental (ex: mudança de monolito para microserviços) são `MAJOR`.
- **Processo de Evolução:** Mudanças arquiteturais significativas devem ser propostas através de um **Architectural Decision Record (ADR)**. Um ADR é um documento curto em markdown que descreve o contexto, a decisão tomada e as consequências.
- **Documentação (ADRs):** Será criada uma pasta `docs/adr/` no repositório para armazenar os ADRs, criando um registro histórico das decisões de arquitetura.

---

## 23. Métricas de Qualidade e Quality Gates

- **Cobertura de Código:** Mínimo de **85%** de cobertura de testes para todo o código de negócio. A cobertura será medida com `pytest-cov` e verificada no pipeline de CI.
- **Complexidade Ciclomática:** Nenhuma função/método deve exceder uma complexidade de **10**. Verificado estaticamente pelo `ruff`.
- **Quality Gates Automatizados (no CI):** Um Pull Request só poderá ser mesclado se:
  1.  Todos os testes passarem.
  2.  A cobertura de código não diminuir.
  3.  Nenhum erro de linter (`ruff`, `eslint`) for encontrado.
  4.  Nenhuma vulnerabilidade de segurança de criticidade alta for detectada por ferramentas de SAST (ex: Snyk, CodeQL).

---

## 24. Análise de Riscos e Plano de Mitigação

| Categoria       | Risco Identificado                                                                    | Probabilidade (1-5) | Impacto (1-5) | Score (P×I) | Estratégia de Mitigação                                                                                                                                                           |
| :-------------- | :------------------------------------------------------------------------------------ | :-----------------: | :-----------: | :---------: | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Segurança**   | Vazamento de dados entre tenants devido a falha na lógica de isolamento.              |          2          |       5       |     10      | Middleware mandatório para escopo de tenant, testes de integração para verificar o isolamento em cada endpoint crítico, code reviews focados em segurança.                        |
| **Negócio**     | Inconsistência nos cálculos financeiros (juros, parcelas, saldo devedor).             |          3          |       5       |     15      | Lógica de cálculo centralizada em módulos de domínio puros e bem definidos. Cobertura de testes unitários de 100% para toda a lógica de cálculo, com múltiplos cenários.          |
| **Técnico**     | Acoplamento excessivo entre os módulos do monolito, dificultando a manutenção futura. |          3          |       3       |      9      | Adoção rigorosa da arquitetura em camadas. Uso de interfaces (ABCs) para comunicação entre serviços de diferentes módulos. Revisões de arquitetura periódicas.                    |
| **Performance** | Lentidão em relatórios e listagens com grande volume de dados.                        |          4          |       3       |     12      | Otimização de queries (uso de `select_related`, `prefetch_related`), indexação estratégica de colunas no banco de dados, implementação de paginação em todas as APIs de listagem. |

---

## 25. Conteúdo dos Arquivos de Ambiente e CI/CD

### `pyproject.toml` Proposto

```toml
[tool.poetry]
name = "iabank"
version = "0.1.0"
description = "IABANK Backend API"
authors = ["Your Name <your.email@example.com>"]
readme = "README.md"
packages = [{include = "iabank", from = "src"}]

[tool.poetry.dependencies]
python = "^3.11"
django = "^4.2"
djangorestframework = "^3.14"
psycopg2-binary = "^2.9.9"
django-environ = "^0.11.2"
celery = "^5.3.6"
redis = "^5.0.1"
gunicorn = "^21.2.0"
pydantic = {extras = ["email"], version = "^1.10.13"}
structlog = "^23.2.0"
django-filter = "^23.3"
djangorestframework-simplejwt = "^5.3.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.3"
pytest-django = "^4.7.0"
factory-boy = "^3.3.0"
pytest-cov = "^4.1.0"
black = "^23.11.0"
ruff = "^0.1.6"
pre-commit = "^3.5.0"

[tool.ruff]
line-length = 88
select = ["E", "F", "W", "I", "C90"]

[tool.black]
line-length = 88

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### `.pre-commit-config.yaml` Proposto

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
  - repo: https://github.com/psf/black
    rev: 23.11.0
    hooks:
      - id: black
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.6
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
```

### `Dockerfile` Proposto (Backend)

```dockerfile
# --- Build Stage ---
FROM python:3.11-slim as builder

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

RUN pip install --upgrade pip
RUN pip install poetry

COPY backend/pyproject.toml backend/poetry.lock ./
RUN poetry config virtualenvs.create false && \
    poetry install --no-dev --no-interaction --no-ansi

# --- Final Stage ---
FROM python:3.11-slim

WORKDIR /app

# Create a non-root user
RUN addgroup --system app && adduser --system --group app

COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY backend/src/ .

USER app

EXPOSE 8000

CMD ["gunicorn", "iabank.wsgi:application", "--bind", "0.0.0.0:8000"]
```

### `Dockerfile` Proposto (Frontend)

```dockerfile
# --- Build Stage ---
FROM node:18-alpine as builder

WORKDIR /app

COPY frontend/package.json frontend/pnpm-lock.yaml ./
RUN npm install -g pnpm
RUN pnpm install

COPY frontend/ .
RUN pnpm build

# --- Final Stage ---
FROM nginx:1.25-alpine

COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx config if you have one
# COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

---

## 26. Estratégia de Processamento Assíncrono (Celery)

Tarefas assíncronas são cruciais para a performance e responsividade do sistema. A estratégia para o uso de Celery foca em robustez, confiabilidade e monitoramento.

- **Idempotência:**

  - **Princípio:** Toda tarefa que modifica dados deve ser idempotente, ou seja, executá-la múltiplas vezes com os mesmos parâmetros deve produzir o mesmo resultado que executá-la uma única vez.
  - **Estratégia:** Tarefas críticas (ex: processar um pagamento) utilizarão um ID de idempotência único. A lógica da tarefa irá primeiro verificar se uma operação com aquele ID já foi concluída antes de prosseguir.

- **Retentativas e Backoff Exponencial:**

  - **Princípio:** Tarefas que dependem de serviços externos (APIs, envio de e-mail) podem falhar por motivos transitórios. A tarefa não deve falhar imediatamente, mas tentar novamente.
  - **Estratégia:** Utilizaremos o mecanismo de retentativa automática do Celery com backoff exponencial. Isso significa que o intervalo entre as tentativas aumentará, evitando sobrecarregar um serviço que já está instável.
  - **Exemplo:**

    ```python
    from celery import shared_task
    @shared_task(
        autoretry_for=(Exception,),
        retry_kwargs={'max_retries': 5},
        retry_backoff=True
    )
    def send_notification_task(user_id, message):
        # Lógica de envio que pode falhar
        pass
    ```

- **Confirmação de Tarefa (Acks Late):**

  - **Princípio:** Por padrão, o broker (Redis/RabbitMQ) considera uma tarefa como "entregue" no momento em que um worker a consome. Se o worker falhar no meio da execução, a tarefa é perdida.
  - **Estratégia:** Todas as tarefas críticas serão configuradas com `acks_late = True`. Isso faz com que a confirmação seja enviada ao broker apenas **após** a tarefa ser concluída com sucesso. Se o worker falhar, o broker re-enfileirará a tarefa para outro worker executar.

- **Filas de "Letras Mortas" (Dead-Letter Queues):**
  - **Princípio:** Algumas tarefas falharão consistentemente mesmo após todas as retentativas (ex: devido a um bug ou dados inválidos). Elas não devem permanecer na fila principal para sempre.
  - **Estratégia:** Será configurado um mecanismo de Dead-Letter Queue (DLQ) no broker (RabbitMQ). Após o número máximo de retentativas, a tarefa e seu contexto de erro serão movidos para uma fila separada. Isso permite que a equipe de desenvolvimento analise a falha manualmente sem interromper o processamento das tarefas válidas.

---

## 27. Plano de Recuperação de Desastres (Disaster Recovery - DR)

Além dos backups operacionais, um plano de DR garante a continuidade do negócio em caso de uma falha catastrófica em larga escala (ex: indisponibilidade de uma região inteira do provedor de nuvem).

- **Objetivo:** Restaurar a funcionalidade completa do serviço em uma região geográfica secundária.
- **Estratégia: Pilot Light**

  - **Banco de Dados:** O banco de dados PostgreSQL será configurado com replicação assíncrona contínua para uma instância "standby" em uma região de DR. Esta instância réplica permanecerá "fria", mas pronta para ser promovida a primária.
  - **Imagens de Contêiner:** As imagens Docker da aplicação serão armazenadas em um registro geo-replicado (ex: AWS ECR com replicação inter-regional).
  - **Infraestrutura como Código (IaC):** Toda a infraestrutura (redes, balanceadores de carga, serviços de aplicação) é definida usando Terraform. Isso permite recriar rapidamente toda a stack de aplicação na região de DR.
  - **Processo de Failover:** Em caso de desastre, um processo semi-automatizado será acionado:
    1. Promover a réplica do banco de dados na região de DR para se tornar a instância primária.
    2. Executar os scripts de IaC para provisionar a infraestrutura da aplicação na região de DR.
    3. Atualizar os registros de DNS para apontar o tráfego para o novo endpoint na região de DR.

- **Testes de DR:** O plano de DR será testado anualmente para garantir sua eficácia e para treinar a equipe no procedimento de failover.
