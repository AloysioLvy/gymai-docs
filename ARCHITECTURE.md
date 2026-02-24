# Diagrama de Arquitetura e Fluxo — GymAI

---

## 1. Visão Geral dos Componentes

```mermaid
graph TB
    subgraph Browser["🌐 Browser"]
        UI["Next.js Frontend\n:3000"]
    end

    subgraph BFF["🔀 BFF — Next.js API Routes"]
        AUTH["WorkOS AuthKit\n(sessão + OAuth)"]
        R1["/api/workout/generate"]
        R2["/api/workout/user/:userId"]
        R3["/api/gym-profile"]
        R4["/api/exercises/search"]
    end

    subgraph Backend["⚙️ Fastify Backend :3001"]
        WC["WorkoutController"]
        GPC["GymProfileController"]
        EC["ExerciseController"]
        UC["UserController"]
    end

    subgraph UseCases["📦 Use Cases (Application Layer)"]
        GWU["GenerateWorkoutUseCase"]
        EWV["EnrichWorkoutWithVideosUseCase"]
        UGPU["UpdateGymProfileUseCase"]
        SEU["SearchExerciseUseCase"]
        CUU["CreateUserUseCase"]
    end

    subgraph External["☁️ Serviços Externos"]
        GPT["OpenAI GPT-4o\n(via LangChain)"]
        YT["YouTube Data API v3"]
        EDB["ExerciseDB API"]
        WOS["WorkOS\n(OAuth / SSO)"]
    end

    subgraph DB["🗄️ PostgreSQL :5432"]
        T1["users"]
        T2["gym_profiles"]
        T3["workouts"]
        T4["workout_exercises"]
        T5["exercises"]
        T6["exercise_video_cache"]
    end

    UI --> BFF
    BFF --> AUTH
    AUTH --> WOS
    BFF --> Backend
    Backend --> WC & GPC & EC & UC
    WC --> GWU
    GWU --> EWV
    GPC --> UGPU
    EC --> SEU
    UC --> CUU
    GWU --> GPT
    EWV --> YT
    SEU --> EDB
    GWU & UGPU & SEU & CUU & EWV --> DB
```

---

## 2. Jornada do Usuário (Fluxo Principal)

```mermaid
flowchart TD
    Start([Usuário acessa a aplicação]) --> Login

    Login{Está autenticado?}
    Login -- Não --> WorkOS["Redireciona para\n/login WorkOS"]
    WorkOS --> OAuth["OAuth SSO\n(Google, GitHub, etc.)"]
    OAuth --> Callback["/callback — troca code por sessão"]
    Callback --> Login

    Login -- Sim --> SyncUser["POST /api/users\nCria ou recupera usuário no banco"]
    SyncUser --> CheckProfile{Tem GymProfile?}

    CheckProfile -- Não --> Onboarding
    CheckProfile -- Sim --> Dashboard

    subgraph Onboarding["📋 Onboarding (formulário multi-etapas)"]
        direction TB
        Q1["Objetivo (hipertrofia, emagrecimento...)"]
        Q2["Nível de fitness"]
        Q3["Dados físicos (idade, peso, altura)"]
        Q4["Dias/semana + duração da sessão"]
        Q5["Equipamentos disponíveis"]
        Q6["Preferências de cardio"]
        Q7["Lesões e condições de saúde"]
        Q8["Grupos musculares foco"]
        Q1 --> Q2 --> Q3 --> Q4 --> Q5 --> Q6 --> Q7 --> Q8
        Q8 --> SaveProfile["POST /api/gym-profile\nSalva respostas em JSON"]
    end

    SaveProfile --> Dashboard

    subgraph Dashboard["🏋️ WorkoutDashboard"]
        direction TB
        LoadWorkouts["Carrega treinos do usuário\nGET /api/workout/user/:userId"]
        Views{Escolhe view}
        LoadWorkouts --> Views
        Views -- Dashboard --> ViewWorkout["Exibe WorkoutResult\n(treino selecionado)"]
        Views -- Busca --> Search["Busca de Exercícios"]
        ViewWorkout --> GenBtn{Clica em\nGerar Treino?}
        GenBtn -- Sim, limite < 2 --> Generate["Fluxo de Geração"]
        GenBtn -- Limite atingido\n2 treinos --> Blocked["Botão desabilitado\nMensagem informativa"]
    end
```

---

## 3. Fluxo de Geração de Treino (Detalhado)

```mermaid
sequenceDiagram
    actor User as Usuário
    participant FE as Next.js Frontend
    participant BFF as BFF /api/workout/generate
    participant BE as Fastify Backend
    participant UC as GenerateWorkoutUseCase
    participant DB as PostgreSQL
    participant AI as OpenAI GPT-4o
    participant YT as YouTube API

    User->>FE: Clica "Gerar Treino"
    FE->>BFF: POST /api/workout/generate

    BFF->>BFF: withAuth() — extrai userId da sessão WorkOS
    alt Não autenticado
        BFF-->>FE: 401 Unauthorized
    end

    BFF->>BE: POST /workout/generate {userId}
    Note over BFF,BE: Header x-internal-secret

    BE->>UC: execute({ userId })

    UC->>DB: findByUserId → GymProfile
    alt GymProfile não encontrado
        UC-->>BE: Error 404
        BE-->>BFF: 404 Not Found
        BFF-->>FE: Erro exibido ao usuário
    end

    UC->>DB: findByUserId → Workouts (count)
    alt Já tem 2 treinos
        UC-->>BE: Error 403
        BE-->>BFF: 403 Forbidden
        BFF-->>FE: Botão bloqueado / mensagem
    end

    UC->>AI: generateWorkoutPlan(gymProfile)
    Note over AI: GPT-4o recebe answers do perfil\nResponde plano semanal em português\nSaída estruturada (Zod schema)
    AI-->>UC: JSON com dias, exercícios, séries, reps

    UC->>UC: EnrichWorkoutWithVideosUseCase
    loop Para cada exercício
        UC->>YT: searchExerciseVideo(exerciseName)
        alt YouTube quota OK
            YT-->>UC: { videoId, title, url }
        else Quota excedida (429)
            YT-->>UC: null (segue sem vídeo)
        end
    end

    UC->>DB: workout.create()\nSalva Workout + WorkoutExercises
    DB-->>UC: Workout salvo

    UC-->>BE: WorkoutResponseDto
    BE-->>BFF: 200 OK + dados do treino
    BFF-->>FE: Novo treino
    FE->>FE: Atualiza estado\nExibe WorkoutResult
    FE-->>User: Plano de treino exibido
```

---

## 4. Fluxo de Busca de Exercícios

```mermaid
sequenceDiagram
    actor User as Usuário
    participant FE as Next.js Frontend
    participant BFF as BFF /api/exercises/search
    participant BE as Fastify Backend
    participant DB as PostgreSQL (cache)
    participant EDB as ExerciseDB API

    User->>FE: Digita nome do exercício
    FE->>BFF: GET /api/exercises/search?q=supino

    BFF->>BE: GET /exercises/supino
    Note over BFF,BE: Rota pública — sem x-internal-secret

    BE->>DB: SELECT exercises WHERE name LIKE '%supino%'

    alt Cache hit (encontrou no banco)
        DB-->>BE: Lista de exercícios
    else Cache miss
        BE->>EDB: GET exercisedb.dev/api/v1/exercises/search?q=supino
        EDB-->>BE: Exercícios com GIF, músculos, instruções
        BE->>DB: INSERT exercícios (cache)
        DB-->>BE: Ok
    end

    BE-->>BFF: JSON com exercícios
    BFF-->>FE: Resultados
    FE-->>User: ExerciseList com GIFs, músculos, instruções
```

---

## 5. Camadas da Aplicação (Backend DDD)

```mermaid
graph LR
    subgraph Interface["Interface Layer\n(HTTP)"]
        Routes["Fastify Routes"]
        Controllers["Controllers"]
    end

    subgraph Application["Application Layer"]
        UseCases["Use Cases"]
        DTOs["DTOs (Input/Output)"]
        Providers["Provider Interfaces"]
    end

    subgraph Domain["Domain Layer"]
        Aggregates["Aggregates\n(User, Workout, GymProfile)"]
        Repositories["Repository Interfaces"]
    end

    subgraph Infrastructure["Infrastructure Layer"]
        PrismaRepos["Prisma Repositories"]
        LangchainProvider["LangChain Provider\n(GPT-4o)"]
        YoutubeProvider["YouTube Provider"]
        PrismaClient["Prisma Client"]
    end

    subgraph DI["IoC Container\n(Inversify)"]
        Container["container.ts\nbindings"]
    end

    Routes --> Controllers
    Controllers --> UseCases
    UseCases --> Repositories
    UseCases --> Providers
    Repositories -.implementado por.-> PrismaRepos
    Providers -.implementado por.-> LangchainProvider
    Providers -.implementado por.-> YoutubeProvider
    PrismaRepos --> PrismaClient
    DI -.injeta.-> Controllers
    DI -.injeta.-> UseCases
    DI -.injeta.-> PrismaRepos
```

---

## 6. Modelo de Dados

```mermaid
erDiagram
    User {
        string id PK
        string email UK
        string firstName
        string lastName
        string phone
        datetime createdAt
        datetime updatedAt
    }

    GymProfile {
        string id PK
        string userId FK
        json answers
        string formVersion
        datetime createdAt
        datetime updatedAt
    }

    Workout {
        string id PK
        string name
        string userId FK
        string status
        json aiOutput
        datetime createdAt
        datetime updatedAt
    }

    WorkoutExercise {
        string id PK
        string workoutId FK
        string exerciseId FK
        int order
        int sets
        int reps
        int restSeconds
        string tempo
        float loadKg
        float rpe
        string notes
    }

    Exercise {
        string id PK
        string name
        string gifUrl
        string bodyPart
        string target
        json secondaryMuscles
        json instructions
        string equipment
    }

    ExerciseVideoCache {
        string id PK
        string exerciseId FK
        string videoId
        string title
        string url
        datetime cachedAt
    }

    User ||--o| GymProfile : "tem"
    User ||--o{ Workout : "gera (máx. 2)"
    Workout ||--o{ WorkoutExercise : "contém"
    Exercise ||--o{ WorkoutExercise : "referenciado em"
    Exercise ||--o| ExerciseVideoCache : "tem vídeo"
```

---

## Resumo do Fluxo em Texto

```
1. LOGIN
   Usuário → WorkOS OAuth → sessão criada → usuário sincronizado no banco

2. ONBOARDING (apenas novos usuários)
   Formulário 8 etapas → respostas salvas como JSON no GymProfile

3. DASHBOARD
   Treinos carregados do banco → seleção e visualização

4. GERAR TREINO
   [BFF] Autentica sessão → extrai userId seguro
   [Backend] Valida GymProfile existe
   [Backend] Valida limite ≤ 2 treinos
   [GPT-4o] Gera plano semanal personalizado em PT-BR
   [YouTube] Enriquece cada exercício com link de vídeo
   [PostgreSQL] Persiste treino + exercícios
   [Frontend] Exibe resultado imediatamente

5. BUSCAR EXERCÍCIO
   Query → cache PostgreSQL (ExerciseDB) → retorna GIF + músculos + instruções
```
