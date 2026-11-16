# PROGRESS.md — прогресс обучения Java Backend

> Правила общения и формат обучения описаны в инструкциях проекта.
> Этот файл — только **состояние обучения**: что пройдено, где остановились, что дальше.
> Обновляется в конце каждой темы.

---

## ▶️ Текущая точка (начинать отсюда)

**Блок:** Spring Security → авторизация на уровне конкретного ресурса.

**Вопрос, на который я ещё не ответил:**

> Сервис сначала делает `SELECT` и проверяет, что файл принадлежит пользователю,
> а затем отдельным запросом выполняет `DELETE`. Почему проверка и удаление
> двумя отдельными SQL-запросами могут создать проблему?
> *Подсказка: атомарность и конкурентный доступ.*

**Что разобрать после ответа:**
1. TOCTOU (time-of-check to time-of-use) / race condition простыми словами.
2. Способы защиты: один условный `DELETE ... WHERE id = ? AND owner_id = ?`,
   транзакции и уровни изоляции, блокировки (`SELECT ... FOR UPDATE`) — без перегруза.
3. Как совмещать общие правила Spring Security и проверку владельца в сервисе.
4. Роли и authorities из БД: таблицы `users`, `roles`, `user_roles`, загрузка через JPA
   в `UserDetails`.
5. Закрепление: интеграция с PostgreSQL / Spring Data JPA + тесты.

---

## 🗺️ Roadmap

| # | Этап | Статус |
|---|------|--------|
| 1 | Java Core, ООП, коллекции, Maven | ✅ |
| 2 | JDBC, Servlet API, PostgreSQL | ✅ |
| 3 | Слои Controller / Service / Repository | ✅ |
| 4 | JUnit 5, Mockito, AssertJ | ✅ |
| 5 | Spring Core (DI/IoC), Spring MVC, Thymeleaf | ✅ |
| 6 | Hibernate, связи сущностей, транзакции | ✅ |
| 7 | Проект «Табло теннисного матча» | ✅ v1, идёт рефакторинг |
| 8 | Spring Data JPA | ⏸️ начат, отложен ради Security |
| 9 | **Spring Security** | 🔄 **в процессе** |
| 10 | Проект «Облачное хранилище файлов» | ⏳ следующий |

### Проект «Табло теннисного матча»
- Стек: Java, Spring Framework 7, Spring MVC, Hibernate 7, PostgreSQL, Thymeleaf.
- **Без Spring Boot** — специально, чтобы понять ручную конфигурацию Spring MVC.
- Статус: первая рабочая версия готова, идёт рефакторинг и проверка.

### Проект «Облачное хранилище файлов» (следующий)
Многопользовательское хранилище наподобие Google Drive.

- **Функции:** регистрация/вход/выход; загрузка файлов и папок; создание папок;
  удаление, переименование, перемещение; скачивание файлов и папок в ZIP;
  просмотр папок; поиск; запрет доступа к чужим файлам.
- **Стек:** Spring Boot, Spring Security, Spring Session + Redis, REST API, Swagger,
  PostgreSQL, Spring Data JPA, Flyway/Liquibase, MinIO (S3) + MinIO Java SDK,
  React (фронт), JUnit, Testcontainers, Docker, Docker Compose, деплой на Linux.
- **Сессии:** два варианта по roadmap — вручную через Redis или
  Spring Security + Spring Session + Redis.

---

## ✅ Spring Security — что уже пройдено

План блока: фильтры → аутентификация/авторизация → `UserDetailsService` и
`PasswordEncoder` → `authorizeHttpRequests` → сессии и cookies → logout и ошибки →
CSRF/CORS → интеграция с PostgreSQL/JPA. Основные пункты пройдены.

### Фильтры и `SecurityFilterChain`
- Запрос проходит цепочку фильтров **до** контроллера.
- Правила доступа к URL задаются через `authorizeHttpRequests`:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/login", "/register").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .requestMatchers("/files/**").authenticated()
            .anyRequest().denyAll())
        .formLogin(Customizer.withDefaults());
    return http.build();
}
```

### Цепочка аутентификации
`AuthenticationManager` → `DaoAuthenticationProvider` → `UserDetailsService` →
`UserRepository` → БД → `PasswordEncoder.matches()`.

| Компонент | Ответственность |
|-----------|-----------------|
| `AuthenticationManager` | Запускает аутентификацию, выбирает подходящий provider |
| `AuthenticationProvider` (`DaoAuthenticationProvider`) | Выполняет конкретную проверку |
| `UserDetailsService` | Загружает пользователя по username |
| `PasswordEncoder` | Проверяет пароль |
| `UserRepository` | Достаёт пользователя из БД |

- `loadUserByUsername()` бросает `UsernameNotFoundException` — это исключение Spring Security.
- `formLogin()` — Spring сам обрабатывает форму входа. Ручной вызов
  `authenticationManager.authenticate()` нужен, например, для своего REST-эндпоинта логина.

### `PasswordEncoder`
- Регистрация → `encode()`, в БД сохраняется **хэш**.
- Вход → `matches(raw, hash)`.
- BCrypt использует **соль**: два `encode()` одного пароля дают разные хэши,
  поэтому строки напрямую не сравнивают.

### `SecurityContext`
- Успешная аутентификация = объект `Authentication` в `SecurityContext`.
- `SecurityContextHolder` — доступ к текущему контексту (по умолчанию `ThreadLocal`).
- `SecurityContextRepository` — загрузка и сохранение контекста **между запросами**
  (например, в HTTP-сессии).
- ⚠️ Просто положить контекст в `SecurityContextHolder` недостаточно: при ручном
  REST-логине контекст нужно явно сохранить через `SecurityContextRepository`.

### CSRF и CORS
- **CORS** — каким origin'ам браузер разрешает обращаться к серверу.
- **CSRF** — подделка запроса, который браузер отправляет с автоматически
  прикреплёнными credentials (cookies).

### Регистрация и сервисный слой
- `UserDetailsService` — только загрузка пользователя для Security.
- `UserService` — бизнес-операции (регистрация и т.п.).
- ⚠️ `existsByUsername()` не защищает от гонки: два запроса могут пройти проверку
  одновременно. Уникальность обеспечивает **`UNIQUE` в БД**, а ошибку нарушения
  ограничения нужно превращать в понятную ошибку приложения.

### Роли и authorities
- Роль — группа полномочий (`USER`, `ADMIN`). `GrantedAuthority` — полномочие,
  которое проверяет Spring Security.
- `hasRole("ADMIN")` проверяет `ROLE_ADMIN` (префикс добавляется сам).
- `hasRole("ROLE_ADMIN")` → проверит `ROLE_ROLE_ADMIN` ❌.
- `hasAuthority("ROLE_ADMIN")` / `hasAuthority("FILE_DELETE")` — точное имя.
- Authorities — для **общих** прав (`FILE_DELETE_ANY`), а не для каждого файла
  (никаких `FILE_125_DELETE`).

### Авторизация на уровне ресурса
- Пример: Алекс (`USER`) пытается удалить файл Боба → проверка владельца в **сервисе**.
- `SecurityConfig` — общие правила по URL и authorities; сервис — бизнес-правила
  (чей это файл).
- ADMIN не получает доступ к чужим файлам автоматически — только при явном
  полномочии (`FILE_DELETE_ANY`).
- Проверки в сервисе и ограничения/условия в БД **дополняют** друг друга.
  Защищать инварианты на разных уровнях — нормально; дублировать всю
  бизнес-логику — нет.

---

## ⚠️ Слабые места и что держать в голове

- Конкурентность: проверка и действие, разнесённые во времени, — источник гонок.
  Тема в процессе.
- Сохранение `SecurityContext` между запросами при ручном логине — повторить на практике.
- Spring Data JPA начат, но не закреплён.

---

## 📚 Документация (Spring Security 7)

- [Servlet Authentication Architecture](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html)
- [DaoAuthenticationProvider](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/dao-authentication-provider.html)
- [Authorize HTTP Requests](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html)
- [Persisting Authentication](https://docs.spring.io/spring-security/reference/servlet/authentication/persistence.html)
