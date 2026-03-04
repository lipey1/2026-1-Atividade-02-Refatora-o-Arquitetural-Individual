# ARCH — Arquitetura após refatoração

## Estrutura final (árvore de pastas relevante)

```text
lib/
  main.dart

  core/
    errors/
      app_error.dart

  features/
    todos/
      domain/
        entities/
          todo.dart
        repositories/
          todo_repository.dart

      data/
        models/
          todo_model.dart
        datasources/
          todo_remote_datasource.dart
          todo_local_datasource.dart
        repositories/
          todo_repository_impl.dart

      presentation/
        app_root.dart
        pages/
          todos_page.dart
        viewmodels/
          todo_viewmodel.dart
        widgets/
          add_todo_dialog.dart
```

Pastas antigas (`models/`, `services/`, `repositories/`, `screens/`, `ui/`, `utils/`, `viewmodels/`, `widgets/`) foram mantidas apenas como *fachadas* (exports) para os novos arquivos em `features/`, para não quebrar imports existentes e deixar clara a migração para feature-first.

## Fluxo de dependências

Fluxo principal da feature `todos`:

1. **UI (Presentation)**
   - `main.dart` cria o `MultiProvider` e registra `TodoViewModel`.
   - `AppRoot` (`features/todos/presentation/app_root.dart`) monta o `MaterialApp` e define `TodosPage` como tela inicial.
   - `TodosPage` (`presentation/pages/todos_page.dart`) exibe a lista de TODOs, dispara `loadTodos`, `addTodo` e `toggleCompleted` chamando apenas métodos do `TodoViewModel`.
   - `AddTodoDialog` (`presentation/widgets/add_todo_dialog.dart`) apenas coleta o texto do usuário e retorna a string para a página.

2. **ViewModel**
   - `TodoViewModel` (`presentation/viewmodels/todo_viewmodel.dart`) expõe estado observável (`items`, `isLoading`, `errorMessage`, `lastSyncLabel`) e orquestra os casos de uso chamando o **repositório** por meio da abstração `TodoRepository`.
   - Não conhece `BuildContext` (apenas é usado pela UI) nem nenhum `Widget`; toda comunicação com a UI é feita via mudança de estado + `notifyListeners()`.

3. **Repository**
   - `TodoRepository` (`domain/repositories/todo_repository.dart`) define a interface (`fetchTodos`, `addTodo`, `updateCompleted`) e o DTO de retorno `TodoFetchResult`.
   - `TodoRepositoryImpl` (`data/repositories/todo_repository_impl.dart`) centraliza a decisão entre fontes **remota** e **local**:
     - Sempre busca a lista em `TodoRemoteDataSource`.
     - Atualiza a informação de última sincronização em `TodoLocalDataSource`.
     - Converte `TodoModel` (data) em `Todo` (domain) antes de devolver para o ViewModel.

4. **DataSources**
   - `TodoRemoteDataSource` (`data/datasources/todo_remote_datasource.dart`) encapsula todo acesso HTTP (`package:http`), endpoints e parsing JSON.
   - `TodoLocalDataSource` (`data/datasources/todo_local_datasource.dart`) encapsula todo uso de `SharedPreferences` para salvar/buscar a última sincronização.

Dependência em camadas:

`UI (presentation)` → `ViewModel` → `TodoRepository` → `TodoRepositoryImpl` → (`TodoRemoteDataSource`, `TodoLocalDataSource`)

## Decisões de responsabilidade

- **Validação**
  - Regras de validação de entrada do usuário (por exemplo, título não vazio ao adicionar TODO) ficam no `TodoViewModel`.
  - A UI apenas envia o texto e reage a mensagens de erro expostas pelo ViewModel.

- **Parsing JSON / acesso remoto**
  - Parsing JSON e mapeamento de resposta HTTP para modelos concretos é responsabilidade do `TodoRemoteDataSource` e do `TodoModel` (`fromJson`/`toJson`).
  - O repositório converte `TodoModel` em `Todo` (entidade de domínio) antes de retornar para camadas superiores.

- **Persistência local / SharedPreferences**
  - Toda interação com `SharedPreferences` está isolada em `TodoLocalDataSource`.
  - O repositório apenas chama `saveLastSync` / `getLastSync` para obter/atualizar a informação de sincronização.

- **Tratamento de erros**
  - `TodoRemoteDataSource` valida códigos HTTP e lança `Exception` em caso de falha.
  - `TodoLocalDataSource` trata ausência/parse inválido de datas retornando `null`.
  - O `TodoViewModel` captura exceções dos métodos do repositório e:
    - Atualiza `errorMessage` com mensagens de erro amigáveis.
    - Faz rollback de alterações otimistas ao alternar `completed` em caso de falha ao atualizar no servidor.

- **Isolamento de camadas**
  - A **UI** não acessa diretamente HTTP nem `SharedPreferences`; todo acesso passa por ViewModel → Repository → DataSources.
  - O **ViewModel** não conhece `Widget`, `BuildContext` ou classes de infraestrutura: lida apenas com `Todo`, `TodoRepository` e tipos simples.
  - O **Repository** é o único ponto que conhece e combina **RemoteDataSource** e **LocalDataSource**, permitindo trocar estratégias sem afetar UI/VM.

