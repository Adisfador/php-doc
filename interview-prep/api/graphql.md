# GraphQL

Полный разбор GraphQL: schema, queries, mutations, subscriptions, Laravel Lighthouse, best practices.

---

## 🎯 Что такое GraphQL?

**GraphQL** - язык запросов для API, разработанный Facebook (2015).

**Vs REST:**

| Aspect | REST | GraphQL |
|--------|------|---------|
| **Endpoints** | Много (`/users`, `/posts`, `/comments`) | Один (`/graphql`) |
| **Data Fetching** | Over-fetching/Under-fetching | Клиент запрашивает ровно то, что нужно |
| **Versioning** | `/api/v1`, `/api/v2` | Не нужно (deprecation fields) |
| **Documentation** | Swagger/OpenAPI | Introspection встроено |

**Пример проблемы REST:**

```
GET /users/1
{
  "id": 1,
  "name": "John",
  "email": "john@example.com",
  "phone": "...",      // ← не нужно
  "address": {...},    // ← не нужно
  "created_at": "..."  // ← не нужно
}

GET /users/1/posts   // ← второй запрос
[{...}, {...}]
```

**GraphQL решение:**

```graphql
query {
  user(id: 1) {
    name
    email
    posts {
      title
      content
    }
  }
}

# Один запрос, только нужные поля
```

---

## 📐 Schema (Схема)

**Schema** - контракт между клиентом и сервером.

### Scalar Types

```graphql
# Встроенные типы
Int       # Целое число 32-bit
Float     # Число с плавающей точкой
String    # Строка UTF-8
Boolean   # true/false
ID        # Уникальный идентификатор (string или int)
```

### Object Types

```graphql
type User {
  id: ID!              # ! = non-nullable
  name: String!
  email: String!
  age: Int
  isActive: Boolean!
  posts: [Post!]!      # [Post!]! = non-null array of non-null Posts
  createdAt: DateTime! # Custom scalar
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!        # Relation
  comments: [Comment!]!
  publishedAt: DateTime
}

type Comment {
  id: ID!
  content: String!
  author: User!
  post: Post!
}
```

### Queries (Чтение)

```graphql
type Query {
  # Get user by ID
  user(id: ID!): User
  
  # Get all users
  users: [User!]!
  
  # Pagination
  usersPaginated(
    first: Int = 10
    page: Int = 1
  ): UserPaginator!
  
  # Search
  searchUsers(query: String!): [User!]!
  
  # Get post
  post(id: ID!): Post
  posts: [Post!]!
}

type UserPaginator {
  data: [User!]!
  paginatorInfo: PaginatorInfo!
}

type PaginatorInfo {
  count: Int!
  currentPage: Int!
  lastPage: Int!
  total: Int!
}
```

### Mutations (Запись)

```graphql
type Mutation {
  # Create user
  createUser(input: CreateUserInput!): User!
  
  # Update user
  updateUser(id: ID!, input: UpdateUserInput!): User!
  
  # Delete user
  deleteUser(id: ID!): DeleteResult!
  
  # Create post
  createPost(input: CreatePostInput!): Post!
}

input CreateUserInput {
  name: String!
  email: String!
  password: String!
}

input UpdateUserInput {
  name: String
  email: String
  password: String
}

type DeleteResult {
  success: Boolean!
  message: String
}
```

### Subscriptions (Real-time)

```graphql
type Subscription {
  # Подписка на новые посты
  postCreated: Post!
  
  # Подписка на комментарии поста
  commentAdded(postId: ID!): Comment!
  
  # Подписка на изменения пользователя
  userUpdated(userId: ID!): User!
}
```

### Custom Scalars

```graphql
scalar DateTime
scalar Email
scalar URL
scalar JSON

type User {
  email: Email!
  website: URL
  createdAt: DateTime!
  metadata: JSON
}
```

### Enums

```graphql
enum UserRole {
  ADMIN
  MODERATOR
  USER
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

type User {
  role: UserRole!
}

type Post {
  status: PostStatus!
}
```

### Interfaces

```graphql
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String!
}

type Post implements Node {
  id: ID!
  title: String!
}

# Query
type Query {
  node(id: ID!): Node
}
```

### Unions

```graphql
union SearchResult = User | Post | Comment

type Query {
  search(query: String!): [SearchResult!]!
}

# Client query
query {
  search(query: "john") {
    ... on User {
      name
      email
    }
    ... on Post {
      title
      content
    }
    ... on Comment {
      content
    }
  }
}
```

---

## 🚀 Laravel Lighthouse

**Lighthouse** - лучший GraphQL для Laravel.

### Установка

```bash
composer require nuwave/lighthouse

# Публикация конфига
php artisan vendor:publish --tag=lighthouse-schema

# Публикация дефолтной схемы
php artisan vendor:publish --tag=lighthouse-config
```

### Схема

**graphql/schema.graphql:**

```graphql
"A datetime string with format `Y-m-d H:i:s`, e.g. `2018-05-23 13:43:32`."
scalar DateTime @scalar(class: "Lighthouse\\Schema\\Types\\Scalars\\DateTime")

type Query {
  users: [User!]! @all
  user(id: ID! @eq): User @find
  posts: [Post!]! @all
  post(id: ID! @eq): Post @find
}

type Mutation {
  createUser(input: CreateUserInput! @spread): User! @create
  updateUser(id: ID!, input: UpdateUserInput! @spread): User! @update
  deleteUser(id: ID!): User @delete
  
  createPost(input: CreatePostInput! @spread): Post! @create
}

type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]! @hasMany
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User! @belongsTo
  comments: [Comment!]! @hasMany
  createdAt: DateTime!
}

type Comment {
  id: ID!
  content: String!
  author: User! @belongsTo
  post: Post! @belongsTo
}

input CreateUserInput {
  name: String! @rules(apply: ["required", "string", "max:255"])
  email: String! @rules(apply: ["required", "email", "unique:users,email"])
  password: String! @rules(apply: ["required", "min:8"]) @hash
}

input UpdateUserInput {
  name: String @rules(apply: ["string", "max:255"])
  email: String @rules(apply: ["email", "unique:users,email"])
  password: String @rules(apply: ["min:8"]) @hash
}

input CreatePostInput {
  title: String! @rules(apply: ["required", "max:255"])
  content: String! @rules(apply: ["required"])
  author: BelongsTo!
}

input BelongsTo {
  connect: ID!
}
```

### Directives (Директивы)

**@all** - получить все записи:
```graphql
type Query {
  users: [User!]! @all
}
```

**@find** - найти по ID:
```graphql
type Query {
  user(id: ID! @eq): User @find
}
```

**@create, @update, @delete**:
```graphql
type Mutation {
  createUser(input: CreateUserInput! @spread): User! @create
  updateUser(id: ID!, input: UpdateUserInput! @spread): User! @update
  deleteUser(id: ID!): User @delete
}
```

**@paginate** - пагинация:
```graphql
type Query {
  users(first: Int! = 15, page: Int): UserPaginator! @paginate
}
```

**@hasMany, @belongsTo** - Eloquent relations:
```graphql
type User {
  posts: [Post!]! @hasMany
}

type Post {
  author: User! @belongsTo
}
```

**@can** - authorization:
```graphql
type Mutation {
  updatePost(id: ID!, input: UpdatePostInput!): Post 
    @can(ability: "update", find: "id")
    @update
}
```

**@guard** - authentication:
```graphql
type Query {
  me: User @guard @auth
}
```

**@rules** - validation:
```graphql
input CreateUserInput {
  email: String! @rules(apply: ["required", "email", "unique:users"])
}
```

**@hash** - hash password:
```graphql
input CreateUserInput {
  password: String! @hash
}
```

**@inject** - inject values:
```graphql
type Mutation {
  createPost(input: CreatePostInput!): Post
    @inject(context: "user.id", name: "user_id")
    @create
}
```

### Custom Resolvers

**Когда @all/@find не хватает:**

```graphql
type Query {
  searchUsers(query: String!): [User!]!
}
```

**app/GraphQL/Queries/SearchUsers.php:**

```php
namespace App\GraphQL\Queries;

class SearchUsers
{
    public function __invoke($rootValue, array $args)
    {
        return \App\Models\User::where('name', 'like', "%{$args['query']}%")
            ->orWhere('email', 'like', "%{$args['query']}%")
            ->get();
    }
}
```

**Регистрация в schema:**

```graphql
type Query {
  searchUsers(query: String!): [User!]! 
    @field(resolver: "App\\GraphQL\\Queries\\SearchUsers")
}
```

### Field Resolvers

**Кастомная логика для поля:**

```graphql
type User {
  id: ID!
  name: String!
  fullName: String! @field(resolver: "App\\GraphQL\\Resolvers\\UserFullName")
}
```

```php
namespace App\GraphQL\Resolvers;

use App\Models\User;

class UserFullName
{
    public function __invoke(User $user): string
    {
        return "{$user->first_name} {$user->last_name}";
    }
}
```

### N+1 Problem Solution

**Lighthouse автоматически решает N+1 с Eloquent:**

```graphql
query {
  posts {
    title
    author {  # Eager loading автоматически
      name
    }
  }
}
```

**Эквивалент в Eloquent:**
```php
Post::with('author')->get();
```

---

## 🔍 Queries (Запросы)

### Базовый запрос

```graphql
query {
  user(id: 1) {
    name
    email
  }
}

# Response
{
  "data": {
    "user": {
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
}
```

### Вложенные запросы

```graphql
query {
  user(id: 1) {
    name
    posts {
      title
      comments {
        content
        author {
          name
        }
      }
    }
  }
}
```

### Аргументы

```graphql
query {
  posts(
    first: 10
    orderBy: {field: "created_at", order: DESC}
  ) {
    data {
      title
      createdAt
    }
  }
}
```

### Aliases

```graphql
query {
  adminUser: user(id: 1) {
    name
  }
  normalUser: user(id: 2) {
    name
  }
}

# Response
{
  "data": {
    "adminUser": {"name": "Admin"},
    "normalUser": {"name": "User"}
  }
}
```

### Fragments

```graphql
fragment UserFields on User {
  id
  name
  email
}

query {
  user1: user(id: 1) {
    ...UserFields
  }
  user2: user(id: 2) {
    ...UserFields
  }
}
```

### Variables

```graphql
query GetUser($userId: ID!) {
  user(id: $userId) {
    name
    email
  }
}

# Variables
{
  "userId": 1
}
```

### Directives

**@include** - условное включение:
```graphql
query GetUser($withEmail: Boolean!) {
  user(id: 1) {
    name
    email @include(if: $withEmail)
  }
}
```

**@skip** - условное исключение:
```graphql
query GetUser($skipEmail: Boolean!) {
  user(id: 1) {
    name
    email @skip(if: $skipEmail)
  }
}
```

---

## ✍️ Mutations

### Создание

```graphql
mutation {
  createUser(input: {
    name: "John Doe"
    email: "john@example.com"
    password: "secret123"
  }) {
    id
    name
    email
    createdAt
  }
}
```

### Обновление

```graphql
mutation {
  updateUser(
    id: 1
    input: {
      name: "Jane Doe"
    }
  ) {
    id
    name
  }
}
```

### Удаление

```graphql
mutation {
  deleteUser(id: 1) {
    id
    name
  }
}
```

### Создание с relation

```graphql
mutation {
  createPost(input: {
    title: "My Post"
    content: "Content here"
    author: {
      connect: 1  # Connect to existing user
    }
  }) {
    id
    title
    author {
      name
    }
  }
}
```

---

## 🔔 Subscriptions (Laravel WebSockets)

### Schema

```graphql
type Subscription {
  postCreated: Post!
}
```

### Implementation

**app/GraphQL/Subscriptions/PostCreated.php:**

```php
namespace App\GraphQL\Subscriptions;

use App\Models\Post;
use Illuminate\Http\Request;
use Nuwave\Lighthouse\Schema\Subscriptions\Subscriber;
use Nuwave\Lighthouse\Subscriptions\Contracts\SubscriptionExceptionHandler;

class PostCreated
{
    public function authorize(Subscriber $subscriber, Request $request): bool
    {
        return true; // или проверка auth
    }
    
    public function filter(Subscriber $subscriber, Post $post): bool
    {
        return true; // фильтр для конкретного подписчика
    }
}
```

### Broadcasting

```php
use Nuwave\Lighthouse\Execution\Utils\Subscription;

// В контроллере или событии
Subscription::broadcast('postCreated', $post);
```

### Client (JavaScript)

```javascript
import { createClient } from 'graphql-ws';

const client = createClient({
  url: 'ws://localhost:8000/graphql',
});

const subscription = client.subscribe({
  query: `
    subscription {
      postCreated {
        id
        title
        author {
          name
        }
      }
    }
  `,
}, {
  next: (data) => {
    console.log('New post:', data);
  },
  error: (error) => {
    console.error('Subscription error:', error);
  },
});
```

---

## 🔐 Authentication & Authorization

### Authentication

**config/lighthouse.php:**

```php
'guard' => 'api',
```

**Schema:**

```graphql
type Query {
  me: User! @guard(with: "api") @auth
  
  posts: [Post!]! @guard
}
```

**Custom auth:**

```php
namespace App\GraphQL\Queries;

use Illuminate\Support\Facades\Auth;

class Me
{
    public function __invoke()
    {
        return Auth::user();
    }
}
```

### Authorization (Gates/Policies)

```graphql
type Mutation {
  updatePost(id: ID!, input: UpdatePostInput!): Post
    @can(ability: "update", find: "id")
    @update
    
  deletePost(id: ID!): Post
    @can(ability: "delete", find: "id")
    @delete
}
```

**Policy:**

```php
namespace App\Policies;

use App\Models\User;
use App\Models\Post;

class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }
    
    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->isAdmin();
    }
}
```

---

## 🧪 Testing

### PHPUnit

```php
namespace Tests\Feature;

use Tests\TestCase;
use App\Models\User;

class GraphQLTest extends TestCase
{
    public function test_can_fetch_users()
    {
        User::factory()->count(3)->create();
        
        $response = $this->graphQL('
            query {
                users {
                    id
                    name
                    email
                }
            }
        ');
        
        $response->assertJsonCount(3, 'data.users');
    }
    
    public function test_can_create_user()
    {
        $response = $this->graphQL('
            mutation {
                createUser(input: {
                    name: "John Doe"
                    email: "john@example.com"
                    password: "secret123"
                }) {
                    id
                    name
                    email
                }
            }
        ');
        
        $response->assertJson([
            'data' => [
                'createUser' => [
                    'name' => 'John Doe',
                    'email' => 'john@example.com',
                ]
            ]
        ]);
        
        $this->assertDatabaseHas('users', [
            'email' => 'john@example.com'
        ]);
    }
    
    public function test_requires_authentication()
    {
        $response = $this->graphQL('
            query {
                me {
                    id
                }
            }
        ');
        
        $response->assertGraphQLErrorMessage('Unauthenticated.');
    }
}
```

---

## 🔧 GraphQL Playground

**Доступ:** http://localhost:8000/graphql-playground

**Пример запроса:**

```graphql
# Query
query GetUserWithPosts($userId: ID!) {
  user(id: $userId) {
    name
    email
    posts {
      title
      content
      comments {
        content
        author {
          name
        }
      }
    }
  }
}

# Variables
{
  "userId": 1
}
```

---

## 📊 Best Practices

### 1. Schema Design

**❌ Плохо - слишком детальное:**
```graphql
type User {
  firstName: String
  lastName: String
}
```

**✅ Хорошо - поле + resolver:**
```graphql
type User {
  name: String!
  fullName: String!
}
```

### 2. Pagination

**Всегда используй пагинацию для списков:**

```graphql
type Query {
  users(first: Int = 10, page: Int): UserPaginator! @paginate
}
```

### 3. Error Handling

```graphql
type MutationResponse {
  success: Boolean!
  message: String
  errors: [Error!]
  data: User
}

type Error {
  field: String
  message: String!
}
```

### 4. Versioning

**Не нужно `/v1`, `/v2` - используй deprecation:**

```graphql
type User {
  email: String! @deprecated(reason: "Use `emailAddress` instead")
  emailAddress: String!
}
```

### 5. DataLoader (Batching)

**Lighthouse использует автоматически, но можно кастомизировать:**

```php
namespace App\GraphQL\Resolvers;

use GraphQL\Deferred;

class PostAuthor
{
    private static $userLoader;
    
    public function __invoke($post)
    {
        if (!self::$userLoader) {
            self::$userLoader = app(\Nuwave\Lighthouse\Execution\DataLoader\BatchLoader::class);
        }
        
        return new Deferred(function() use ($post) {
            return self::$userLoader->load($post->user_id);
        });
    }
}
```

---

## 🆚 GraphQL vs REST

| Feature | REST | GraphQL |
|---------|------|---------|
| **Endpoints** | Много | Один `/graphql` |
| **Over-fetching** | Да | Нет |
| **Under-fetching** | Да (N+1 requests) | Нет |
| **Versioning** | `/api/v1` | Deprecation |
| **Documentation** | Swagger | Introspection |
| **Caching** | HTTP cache | Сложнее |
| **Learning Curve** | Низкая | Средняя |

**Когда использовать GraphQL:**
- ✅ Mobile apps (экономия трафика)
- ✅ Сложные данные с relations
- ✅ Разные клиенты (web, mobile, desktop)
- ✅ Real-time subscriptions

**Когда использовать REST:**
- ✅ Простые CRUD
- ✅ Нужен HTTP кэш (Varnish, CDN)
- ✅ Файлы (upload/download)
- ✅ Команда не знакома с GraphQL

---

## 🎓 Для собеседования: ключевые точки

1. **GraphQL** - язык запросов, один endpoint, клиент запрашивает точно нужные данные
2. **Schema** - контракт (Types, Queries, Mutations, Subscriptions)
3. **Queries** - чтение данных
4. **Mutations** - запись данных (create, update, delete)
5. **Subscriptions** - real-time через WebSockets
6. **Lighthouse** - лучший для Laravel, directives @all/@find/@create
7. **N+1** - решается автоматически с Eloquent eager loading
8. **vs REST** - нет over-fetching, один запрос вместо нескольких
9. **Недостатки** - сложнее кэширование, learning curve
10. **Use Case** - mobile apps, сложные relations, real-time

**Главное:** GraphQL позволяет клиенту запрашивать точно нужные данные одним запросом.
