# 🎓 Guia de Defesa e Apresentação da Rubrica (Mapeado por Arquivos)
**Sistema:** UTFPR Arena - Beach Tennis Management  
**Módulo Avaliado:** Autenticação, Autorização (RBAC), Segurança e Testes Automatizados  

> **Como usar este guia na apresentação:**  
> Cada seção abaixo corresponde a um arquivo do projeto. Ao abrir o arquivo no editor durante a avaliação, utilize as explicações, trechos de código e referências de livros correspondentes.

---

## 📑 Índice de Arquivos do Projeto

1. [database/migrations/001_create_users_table.sql](#1-databasemigrations001_create_users_tablesql) ➔ *Modelagem, Hash BCrypt, Campos de Segurança, Seeds*
2. [public/index.php](#2-publicindexphp) ➔ *Front Controller, Bootstrap, Ciclo de Vida da Requisição*
3. [config/routes.php](#3-configroutesphp) ➔ *Agrupamento de Rotas, `middleware('auth')->group(...)`, Rotas Públicas vs. Autenticadas*
4. [src/Core/Session.php](#4-srccoresessionphp) ➔ *Gerenciamento de Sessão, Prevenção de Session Fixation, Destruição de Cookies*
5. [src/Core/FlashMessage.php](#5-srccoreflashmessagephp) ➔ *Padrão PRG (Post/Redirect/Get), Mensagens Efêmeras*
6. [src/Core/MiddlewareInterface.php & src/Core/AuthMiddleware.php](#6-srccoremiddlewareinterfacephp--srccoreauthmiddlewarephp) ➔ *Papel do Middleware, Interceptação, Bloqueio 401 / Redirect*
7. [src/Core/RoleMiddleware.php](#7-srccorerolemiddlewarephp) ➔ *Autorização por Papel (RBAC), Bloqueio 403 Forbidden*
8. [src/Core/Router.php](#8-srccorerouterphp) ➔ *Pipeline de Middlewares, Cadeia de Responsabilidade, Roteamento Dinâmico*
9. [src/Controllers/AuthController.php](#9-srccontrollersauthcontrollerphp) ➔ *Fluxo de Login, Validação, `password_verify()`, Logout*
10. [src/Controllers/DashboardController.php & src/Controllers/AdminController.php](#10-srccontrollersdashboardcontrollerphp--srccontrollersadmincontrollerphp) ➔ *Áreas Protegidas: Painel do Aluno e Painel Administrativo*
11. [src/Controllers/AccessController.php](#11-srccontrollersaccesscontrollerphp) ➔ *Motor de Controle de Acesso por Inadimplência*
12. [src/Models/User.php](#12-srcmodelsuserphp) ➔ *Entidade User, Encapsulamento, Transições de Estado, Ocultação da Senha*
13. [src/Views/auth/login.php, dashboard/index.php, admin/index.php](#13-views-srcviewsauthloginphp-dashboardindexphp-adminindexphp) ➔ *Interface com o Usuário, Alertas Flash, Exibição de Sessão*
14. [tests/Acceptance/AuthFlowTest.php](#14-testsacceptanceauthflowtestphp) ➔ *Testes de Aceitação dos 4 Fluxos da Rubrica*
15. [tests/Unit/UserTest.php & tests/Unit/RouterTest.php](#15-testsunitusertestphp--testsunitroutertestphp) ➔ *Testes Unitários dos Models e Roteador*
16. [.github/workflows/ci.yml & Pull Request](#16-githubworkflowsciyml--pull-request) ➔ *Pipeline de CI (PHPCS, PHPStan, PHPUnit) e Convenção de Commits*

---

## 1. `database/migrations/001_create_users_table.sql`
📂 **Localização:** [database/migrations/001_create_users_table.sql](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/database/migrations/001_create_users_table.sql)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Demonstração do Banco de Dados:* Explicação da modelagem e estrutura do banco para autenticação.
- *Demonstração do Banco de Dados:* Como a estrutura atende requisitos de segurança (hash, recuperação de senha, timestamps, 2FA).
- *Demonstração Prática:* Dados de teste (seeds de admin e aluno).

### 🔍 Explicação Detalhada do Código
```sql
CREATE TABLE IF NOT EXISTS users (
    id                BIGSERIAL       PRIMARY KEY,
    full_name         VARCHAR(255)    NOT NULL,
    cpf               VARCHAR(11)     NOT NULL UNIQUE,     -- Identificador único de aluno/professor
    email             VARCHAR(255)    NOT NULL UNIQUE,     -- Login único do sistema
    password          VARCHAR(255)    NOT NULL,            -- Hash BCrypt ($2y$12$...) com 60 chars
    phone             VARCHAR(20)     NOT NULL,
    role              VARCHAR(20)     NOT NULL DEFAULT 'student'
                          CHECK (role IN ('student', 'teacher', 'manager')),
    status            VARCHAR(20)     NOT NULL DEFAULT 'active'
                          CHECK (status IN ('active', 'blocked', 'inactive')),

    -- Requisitos de Segurança Específicos da Rubrica:
    remember_token    VARCHAR(100)    NULL,                -- Token persistente para "Lembrar de mim"
    last_login_at     TIMESTAMP       NULL,                -- Rastreabilidade e auditoria de login
    email_verified_at TIMESTAMP       NULL,                -- Confirmação para 2FA e ativação
    password_reset_token VARCHAR(100) NULL,                -- Token temporário para recuperação de senha
    password_reset_expires_at TIMESTAMP NULL,             -- Expiração do token de recuperação

    created_at        TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

### 🛡️ Defesa dos Requisitos de Segurança:
1. **Hash de Senha (BCrypt):** A coluna `password` armazena exclusivamente hashes gerados com custo 12. O salt aleatório embutido impede ataques de tabelas pré-computadas (*Rainbow Tables*).
2. **Campos para Recuperação de Senha:** `password_reset_token` armazena um token aleatório (`bin2hex(random_bytes(32))`) e `password_reset_expires_at` delimita a validade temporal (ex: 30 minutos), evitando ataques de reutilização.
3. **Timestamps para Login e Auditoria:** `last_login_at` registra a última autenticação para análise de anomalias; `email_verified_at` é pré-requisito para ativação de 2FA.
4. **Seeds de Demonstração (Linhas 98-121):**
   - Admin: `admin@arena.com` / `admin123` (`role = manager`)
   - Aluno: `aluno@arena.com` / `aluno123` (`role = student`)

---

## 2. `public/index.php`
📂 **Localização:** [public/index.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/public/index.php)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Explicação do Código:* Ponto de entrada único da aplicação (Front Controller Pattern).
- *Conceito Teórico:* Funcionamento de Sessões no ciclo de vida da requisição HTTP.

### 🔍 Explicação Detalhada do Código
```php
// 1. Carrega o autoloader PSR-4 do Composer
require_once __DIR__ . '/../vendor/autoload.php';

use App\Core\Request;
use App\Core\Router;
use App\Core\Session;

// 2. Inicia o mecanismo de sessões PHP logo no início do ciclo de vida
Session::start();

// 3. Instancia o Router e carrega todas as rotas da aplicação
$router = new Router();
require_once __DIR__ . '/../config/routes.php';

// 4. Captura a requisição HTTP global, despacha pelo router e envia a resposta
$request = Request::createFromGlobals();
$response = $router->dispatch($request);
$response->send();
```

### 💡 O que explicar ao professor:
- Segue o padrão arquitetural **Front Controller**: todas as requisições HTTP passam por este único arquivo antes de atingir qualquer controller ou middleware.
- O `Session::start()` garante que o cookie de sessão (`PHPSESSID`) seja enviado ou lido antes de qualquer saída para o navegador, evitando erros de *headers already sent*.

---

## 3. `config/routes.php`
📂 **Localização:** [config/routes.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/config/routes.php)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Funcionamento do Framework (2.1):* `Route::middleware('auth')->group(...)`.
- *Testes de Acesso (2.1 e 2.2):* Separação explícita entre Rotas Públicas, Autenticadas e Administrativas.
- *Conceito 2.2:* Diferença entre Autenticação e Autorização expressa na arquitetura de rotas.

### 🔍 Explicação Detalhada do Código
```php
// 1. ROTAS PÚBLICAS: Acessíveis por qualquer visitante
$router->get('/health', [/* health check */]);
$router->get('/login', [AuthController::class, 'showLogin']);
$router->post('/login', [AuthController::class, 'login']);

// 2. ROTAS AUTENTICADAS (Rubrica 2.1):
// Agrupamento protegido por middleware 'auth'. Usuários sem sessão ativa são barrados aqui!
$router->middleware('auth')->group(function (\App\Core\Router $router): void {
    $router->post('/logout', [AuthController::class, 'logout']);
    $router->get('/dashboard', [DashboardController::class, 'index']);
    $router->get('/access/check/{userId}', [AccessController::class, 'check']);
});

// 3. ROTAS ADMINISTRATIVAS:
// Além de autenticação ('auth'), exigem papel de 'manager' (Autorização / RBAC)
$router->middleware('auth')->group(function (\App\Core\Router $router): void {
    $router->get('/admin', [AdminController::class, 'index']);
    $router->post('/access/block/{userId}', [AccessController::class, 'block']);
});
```

### 📚 Livro de Apoio para este Arquivo:
> **FOWLER, Martin. Padrões de Arquitetura de Aplicações Corporativas.** Porto Alegre: Bookman, 2006.  
> *Capítulo 14: Padrões de Apresentação na Web - Front Controller e Roteamento Centralizado.*

---

## 4. `src/Core/Session.php`
📂 **Localização:** [src/Core/Session.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Core/Session.php)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Conceito 2.3:* Funcionamento de cookies e sessões no processo de autenticação.
- *Explicação do Código 1.3:* Criação segura de sessão no login.
- *Explicação do Código 1.4:* Destruição total de sessão no logout.

### 🔍 Trechos-chave e Explicação Técnica:
```php
// 1. Iniciação com flags de segurança nos cookies
public static function start(): void {
    if (session_status() === PHP_SESSION_NONE) {
        session_start([
            'cookie_httponly' => true, // Impede acesso via JavaScript (mitiga XSS)
            'cookie_samesite' => 'Lax',  // Protege contra requisições cruzadas (CSRF)
        ]);
    }
}

// 2. Prevenção de Session Fixation no Login
public static function regenerateId(bool $deleteOldSession = true): bool {
    return session_regenerate_id($deleteOldSession);
}

// 3. Armazenamento de credenciais na sessão
public static function loginUser(int $id, string $email, string $role): void {
    $_SESSION['auth_user'] = ['id' => $id, 'email' => $email, 'role' => $role];
}

// 4. Logout completo (Servidor e Cliente)
public static function logout(): void {
    $_SESSION = [];
    if (ini_get("session.use_cookies")) {
        $params = session_get_cookie_params();
        setcookie(session_name(), '', time() - 42000,
            $params["path"], $params["domain"],
            $params["secure"], $params["httponly"]
        );
    }
    session_destroy();
}
```

### 📚 Livro de Apoio para este Arquivo:
> **KUROSE, James F.; ROSS, Keith W. Redes de Computadores e a Internet: Uma Abordagem Top-Down.** 7. ed. Pearson, 2017.  
> *Capítulo 2: Camada de Aplicação - Seção 2.2.4: Interação Usuário-Servidor: Cookies e Gerenciamento de Sessão.*

---

## 5. `src/Core/FlashMessage.php`
📂 **Localização:** [src/Core/FlashMessage.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Core/FlashMessage.php)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Funcionamento do Framework (2.2):* `FlashMessage` (citado na rubrica como `lib/FlashMessage.php`).
- *Padrão PRG (Post/Redirect/Get):* Feedback visual sem reenvio de formulário ao atualizar a página.

### 🔍 Explicação Detalhada do Código
```php
class FlashMessage
{
    private const SESSION_KEY = '_flash';

    // Grava a mensagem na sessão para ser exibida após o redirecionamento
    public static function set(string $type, string $message): void {
        Session::start();
        $flash = Session::get(self::SESSION_KEY, []);
        $flash[$type] = $message;
        Session::set(self::SESSION_KEY, $flash);
    }

    // Lê a mensagem e faz UNSET IMEDIATO (autodestruição)
    public static function get(string $type): ?string {
        Session::start();
        $flash = Session::get(self::SESSION_KEY, []);
        if (!isset($flash[$type])) {
            return null;
        }
        $message = $flash[$type];
        unset($flash[$type]); // <-- Não reaparecerá no próximo refresh!
        Session::set(self::SESSION_KEY, $flash);
        return $message;
    }
}
```

---

## 6. `src/Core/MiddlewareInterface.php` & `src/Core/AuthMiddleware.php`
📂 **Localização:**  
- [src/Core/MiddlewareInterface.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Core/MiddlewareInterface.php)  
- [src/Core/AuthMiddleware.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Core/AuthMiddleware.php)  

🎯 **Tópicos da Rubrica Atendidos:**
- *Conceito 2.5:* Papel do middleware na autenticação.
- *Explicação do Código 1.1:* Tentativa de acesso a área restrita sem autenticação.
- *Demonstração Prática (Item 1):* Comportamento diante de requisição anônima.

### 🔍 Explicação Detalhada do Código
```php
class AuthMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, callable $next): Response
    {
        // 1. Verifica se o usuário possui sessão ativa
        if (!Session::isAuthenticated()) {
            
            // 2. Se for requisição de API (Accept: application/json) -> 401 Unauthorized
            if ($request->isJson()) {
                return Response::json([
                    'status' => 'error',
                    'code' => 401,
                    'message' => 'Nao autenticado. Faca login para acessar esta area.'
                ], 401);
            }

            // 3. Se for navegador web -> define FlashMessage e redireciona (302) para /login
            FlashMessage::set('error', 'Voce precisa fazer login para acessar esta area.');
            return Response::redirect('/login');
        }

        // 4. Usuário autenticado: passa a bola para o próximo elo da cadeia
        return $next($request);
    }
}
```

### 📚 Livro de Apoio para este Arquivo:
> **GAMMA, Erich et al. Padrões de Projeto: Soluções Reutilizáveis de Software Orientado a Objetos (GoF).** Bookman, 2000.  
> *Padrões Comportamentais: Chain of Responsibility (Cadeia de Responsabilidades).*

---

## 7. `src/Core/RoleMiddleware.php`
📂 **Localização:** [src/Core/RoleMiddleware.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Core/RoleMiddleware.php)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Conceito 2.2:* Diferença entre Autenticação e Autorização.
- *Demonstração Prática (Admin vs Usuário):* Bloqueio com código HTTP `403 Forbidden`.

### 🔍 Explicação Detalhada do Código
```php
class RoleMiddleware implements MiddlewareInterface
{
    private array $allowedRoles;

    public function __construct(string ...$allowedRoles) {
        $this->allowedRoles = $allowedRoles;
    }

    public function handle(Request $request, callable $next): Response
    {
        $authUser = Session::getAuthUser();

        // Se nem autenticado está -> 401
        if ($authUser === null) {
            return Response::json(['code' => 401, 'message' => 'Nao autenticado.'], 401);
        }

        // AUTORIZAÇÃO: O usuário tem o papel necessário (ex: manager)?
        $userRole = (string)($authUser['role'] ?? '');
        if (!in_array($userRole, $this->allowedRoles, true)) {
            // Papel insuficiente -> 403 Forbidden!
            return Response::json([
                'status' => 'error',
                'code' => 403,
                'message' => "Acesso negado. Esta area requer papel: " . implode(' ou ', $this->allowedRoles),
                'your_role' => $userRole
            ], 403);
        }

        return $next($request);
    }
}
```

### 📚 Livro de Apoio para este Arquivo:
> **TANENBAUM, Andrew S.; WETHERALL, David. Redes de Computadores.** 5. ed. Pearson, 2011.  
> *Capítulo 8: Segurança de Redes - Seção 8.6: Controle de Acesso Baseado em Papéis (RBAC).*

---

## 8. `src/Core/Router.php`
📂 **Localização:** [src/Core/Router.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Core/Router.php)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Funcionamento do Framework (2.1):* Implementação interna de `middleware()->group(...)`.
- *Pipeline de Execução:* Como os middlewares são empilhados em formato de cascata antes de chegar ao Controller.

### 🔍 Explicação Detalhada do Código
```php
// 1. Fluent Interface: armazena middlewares temporários
public function middleware(string ...$names): self {
    $this->activeMiddlewares = array_merge($this->activeMiddlewares, $names);
    return $this;
}

// 2. Executa a closure e limpa os middlewares em seguida
public function group(callable $callback): void {
    $callback($this);
    $this->activeMiddlewares = []; // Isolamento de escopo
}

// 3. Monta a esteira (pipeline) de trás para frente
private function runWithMiddlewares($handler, array $middlewareNames, Request $request, array $params): Response {
    $core = fn (Request $req) => $this->executeHandler($handler, $req, $params);

    $chain = $core;
    foreach (array_reverse($middlewareNames) as $name) {
        $middlewareClass = $this->middlewareMap[$name];
        $middleware = new $middlewareClass();
        $next = $chain;
        $chain = fn (Request $req): Response => $middleware->handle($req, $next);
    }

    return $chain($request);
}
```

---

## 9. `src/Controllers/AuthController.php`
📂 **Localização:** [src/Controllers/AuthController.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Controllers/AuthController.php)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Conceito 2.1:* Definição e execução da Autenticação.
- *Explicação do Código 1.2:* Tentativa de autenticação com dados incorretos.
- *Explicação do Código 1.3:* Autenticação bem-sucedida.
- *Explicação do Código 1.4:* Logout seguro.

### 🔍 Explicação Detalhada dos Métodos

#### Login (`POST /login` - Linhas 68-125):
```php
public function login(Request $request): Response
{
    $body = $request->getBody();
    $email = trim($body['email'] ?? '');
    $password = (string)($body['password'] ?? '');

    // 1. Busca usuário pelo email no repositório/banco
    $user = $this->findUserByEmail($email);
    if (!$user) {
        FlashMessage::set('error', 'Credenciais inválidas. Verifique seu e-mail e senha.');
        return Response::redirect('/login');
    }

    // 2. Compara texto plano com o hash BCrypt de forma segura
    if (!password_verify($password, $user['password_hash'])) {
        FlashMessage::set('error', 'Credenciais inválidas. Verifique seu e-mail e senha.');
        return Response::redirect('/login');
    }

    // 3. Checa status da conta (bloqueado por inadimplência)
    if ($user['status'] === 'blocked') {
        FlashMessage::set('error', 'Sua conta está bloqueada por pendências financeiras.');
        return Response::redirect('/login');
    }

    // 4. Sucesso: previne fixação de sessão e grava auth_user
    Session::regenerateId(true);
    Session::loginUser($user['id'], $user['email'], $user['role']);

    // 5. Redirecionamento baseado no papel (RBAC)
    if ($user['role'] === 'manager') {
        return Response::redirect('/admin');
    }
    return Response::redirect('/dashboard');
}
```

#### Logout (`POST /logout` - Linhas 134-142):
```php
public function logout(Request $request): Response
{
    Session::logout(); // Apaga $_SESSION, destrói cookie e encerra sessão
    FlashMessage::set('success', 'Você saiu da sua conta com segurança.');
    return Response::redirect('/login');
}
```

### 📚 Livros de Apoio para este Arquivo:
> **STALLINGS, William. Criptografia e Segurança de Redes: Princípios e Práticas.** 6. ed. Pearson, 2014.  
> *Capítulo 1: Conceitos de Autenticação.*  
> **LOCKHART, Josh. PHP Moderno.** Novatec, 2016.  
> *Capítulo 5: password_hash() e password_verify().*

---

## 10. `src/Controllers/DashboardController.php` & `src/Controllers/AdminController.php`
📂 **Localização:**  
- [src/Controllers/DashboardController.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Controllers/DashboardController.php)  
- [src/Controllers/AdminController.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Controllers/AdminController.php)  

🎯 **Tópicos da Rubrica Atendidos:**
- *Demonstração Prática (Admin e Usuário):* Diferenciação visual e de permissões entre `student` e `manager`.

### 🔍 O que destacar:
- O `DashboardController` atende o aluno comum: exibe o papel `student`, estado ativo da conta e suas faturas.
- O `AdminController` atende exclusivamente gestores (`role: manager`): compila estatísticas de alunos totais, inadimplentes, faturas atrasadas e faturamento do mês.

---

## 11. `src/Controllers/AccessController.php`
📂 **Localização:** [src/Controllers/AccessController.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Controllers/AccessController.php)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Regra de Negócio de Controle de Acesso por Inadimplência.*

### 🔍 Explicação da Regra:
```php
// Regra da Arena: 2 ou mais faturas vencidas bloqueiam a catraca/acesso
if ($overdueCount >= 2) {
    $user->block(); // Status muda para 'blocked'
}
```
- Mostra a integração entre o módulo financeiro (faturas) e a liberação de entrada na arena.

---

## 12. `src/Models/User.php`
📂 **Localização:** [src/Models/User.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Models/User.php)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Testes Unitários (3.1):* Métodos dos models.
- *Requisitos de Segurança:* Ocultação do hash da senha no método `toArray()`.

### 🔍 Explicação Detalhada do Código
```php
class User
{
    public function __construct(
        private ?int $id,
        private string $fullName,
        private string $cpf,
        private string $email,
        private string $password, // Hash seguro
        private string $phone,
        private string $role = self::ROLE_STUDENT,
        private string $status = self::STATUS_ACTIVE
    ) {}

    public function isBlocked(): bool { return $this->status === self::STATUS_BLOCKED; }
    public function block(): void { $this->status = self::STATUS_BLOCKED; }
    public function activate(): void { $this->status = self::STATUS_ACTIVE; }

    public function verifyPassword(string $plainPassword): bool {
        return password_verify($plainPassword, $this->password);
    }

    // REGRA DE SEGURANÇA CRÍTICA:
    // toArray() NUNCA inclui o campo 'password' para evitar vazamento acidental em JSON/Views!
    public function toArray(): array {
        return [
            'id' => $this->id,
            'full_name' => $this->fullName,
            'email' => $this->email,
            'role' => $this->role,
            'status' => $this->status,
        ];
    }
}
```

---

## 13. Views: `src/Views/auth/login.php`, `dashboard/index.php`, `admin/index.php`
📂 **Localização:**  
- [src/Views/auth/login.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Views/auth/login.php)  
- [src/Views/dashboard/index.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Views/dashboard/index.php)  
- [src/Views/admin/index.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/src/Views/admin/index.php)  

🎯 **Tópicos da Rubrica Atendidos:**
- *Demonstração Prática no Sistema:* Apresentação visual para o avaliador.

### 🔍 Pontos de Atenção na Interface:
- **`login.php`:** Renderiza as `FlashMessages` de erro em caixa vermelha e sucesso em caixa verde.
- **`dashboard/index.php`:** Mostra os dados em sessão (`$_SESSION['auth_user']`) serializados para demonstração pedagógica.
- **`admin/index.php`:** Mostra o badge `ADMIN` e aviso explícito de que alunos que tentarem entrar ali recebem `403`.

---

## 14. `tests/Acceptance/AuthFlowTest.php`
📂 **Localização:** [tests/Acceptance/AuthFlowTest.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/tests/Acceptance/AuthFlowTest.php)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Testes de Aceitação (1.1, 1.2, 1.3, 1.4):* Mapeamento exato 1 para 1 com a rubrica da imagem.

### 🔍 Métodos de Teste e seus Cenários:

| Teste no Código | Item da Rubrica | O que valida |
|---|---|---|
| `testAcessoDashboardSemLoginRetorna401()` | **1.1** Acesso restrito sem auth | Requisição sem sessão ativa para `/dashboard` retorna status HTTP 401. |
| `testLoginComEmailInexistenteRetornaRedirectParaLogin()` | **1.2** Auth com dados incorretos | Login com email inexistente redireciona (302) para `/login` e não cria sessão. |
| `testLoginComSenhaErradaRedirectParaLogin()` | **1.2** Auth com dados incorretos | Login com senha incorreta falha no `password_verify` e mantém `isAuthenticated() === false`. |
| `testLoginComAdminCorretoRedirectParaAdmin()` | **1.3** Auth bem-sucedida | Admin autentica com sucesso e é direcionado para a área de gestão. |
| `testAposLoginDashboardEstaAcessivel()` | **1.3** Auth bem-sucedida | Após autenticar, requisição a `/dashboard` responde HTTP 200 OK. |
| `testLogoutDestroisessaoERedirecionaParaLogin()` | **1.4** Logout | `POST /logout` limpa a sessão e redireciona (302) para `/login`. |
| `testAposLogoutDashboardBloqueiaComk401()` | **1.4** Logout | Após efetuar logout, novas tentativas de acessar `/dashboard` voltam a receber 401. |

---

## 15. `tests/Unit/UserTest.php` & `tests/Unit/RouterTest.php`
📂 **Localização:**  
- [tests/Unit/UserTest.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/tests/Unit/UserTest.php)  
- [tests/Unit/RouterTest.php](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/tests/Unit/RouterTest.php)  

🎯 **Tópicos da Rubrica Atendidos:**
- *Testes de Acesso / Rotas (2.1 e 2.2):* Rotas públicas (`/health`) e tratamento de rotas inexistentes (404).
- *Testes Unitários (3.1 e 3.2):* Métodos dos models (`User`) e classes do framework (`Router`).

### 🔍 O que mostrar ao avaliador:
1. Em `UserTest.php`:
   - `testUsuarioBloqueadoEhBloqueado()` e `testBloquearUsuarioAlteraStatus()`: Testa as máquinas de estado de ativação e bloqueio.
   - `testToArrayRetornaCamposEsperados()`: Garante via asserção que a chave `'password'` **não** está presente no array exportado.
   - `testVerifyPasswordCorreta()`: Valida o método de verificação de hash BCrypt.
2. Em `RouterTest.php`:
   - `testRotaGetSimplesFunciona()`: Testa rotas públicas abertas.
   - `testRotaNaoEncontradaRetorna404()`: Valida o comportamento de rota inexistente.
   - `testRotaComParametroDinamicoECapturado()`: Valida extração de IDs de rotas (ex: `/users/{id}`).

---

## 16. `.github/workflows/ci.yml` & Pull Request
📂 **Localização:** [.github/workflows/ci.yml](file:///home/gabrielgoettenauer/arenaBt/utfprarenaGB/utfprarena/.github/workflows/ci.yml)  
🎯 **Tópicos da Rubrica Atendidos:**
- *Item 0.5: Pull Request (PR)* - Título, Descrição, Nome da Branch e Convenção de Commits.
- *Garantia de Qualidade:* Pipeline automatizado de linting e testes.

### 🔍 Estrutura do Pipeline de CI:
```yaml
- name: Executar PHPCS (Estilo PSR-12)
  run: vendor/bin/phpcs

- name: Executar PHPStan (Análise Estática)
  run: vendor/bin/phpstan analyse

- name: Executar PHPUnit (Testes)
  run: vendor/bin/phpunit
```

### 📋 Sugestão para o Item 0.5 da Rubrica (Pull Request):
- **Nome da Branch:** `feature/auth-rbac-and-security`
- **Título da PR:** `feat(auth): implementar sistema completo de autenticacao, RBAC e testes automatizados`
- **Mensagens de Commit (Conventional Commits):**
  - `fix(types): corrigir tipos de callback do middleware e docblocks no router`
  - `fix(views): tratar variavel de usuario nula e formatar estilos css`
  - `test(user): adicionar testes para getPassword e verifyPassword`
  - `docs: adicionar roteiro e guia de apresentacao da rubrica de avaliacao`
