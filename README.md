# springstarter-auth

`springstarter-auth` est un service d'authentification et d'autorisation développé avec Spring Boot.

Il fournit une authentification par nom d'utilisateur et mot de passe, des jetons d'accès JWT signés avec RSA, des refresh tokens avec rotation et révocation, ainsi qu'une gestion des rôles et permissions persistée dans PostgreSQL.

## Fonctionnalités

- Authentification username/password avec Spring Security
- Mots de passe protégés avec BCrypt
- JWT signés avec RSA
- Claims JWT :
    - `iss`
    - `sub`
    - `aud`
    - `iat`
    - `exp`
    - `jti`
    - `userId`
    - `authorities`
- Refresh tokens opaques et générés avec `SecureRandom`
- Stockage uniquement du hash SHA-256 des refresh tokens
- Rotation des refresh tokens
- Détection des tokens expirés, révoqués ou déjà utilisés
- Déconnexion par révocation du refresh token
- Gestion des utilisateurs, rôles et permissions
- Persistance PostgreSQL
- Gestion du schéma et des données de démonstration avec Liquibase
- Documentation OpenAPI / Swagger UI
- Gestion centralisée des erreurs REST

---

## Architecture

L'application suit une architecture classique en couches :

```text
HTTP
 |
 v
+--------------------------+
| AuthenticationController |
+------------+-------------+
             |
             v
+--------------------------+
|         Services         |
|                          |
| AuthenticationService    |
| RefreshTokenService      |
| UserService              |
+------------+-------------+
             |
             v
+--------------------------+
|       Repositories       |
|                          |
| UserRepository           |
| RoleRepository           |
| RefreshTokenRepository   |
+------------+-------------+
             |
             v
+--------------------------+
|        PostgreSQL        |
|      schéma authdb       |
+--------------------------+
```

Les responsabilités principales sont séparées :

- `AuthenticationService` : authentification et génération des JWT
- `RefreshTokenService` : cycle de vie des refresh tokens
- `UserService` : intégration avec `UserDetailsService`
- `AuthenticationController` : exposition de l'API REST

---

## Modèle d'autorisation

L'application distingue les **rôles** des **permissions**.

Les rôles disponibles sont :

```text
ROLE_USER
ROLE_ADMIN
ROLE_READER
```

Les permissions de démonstration sont :

```text
READ
WRITE
```

Exemple :

```text
admin
 |
 +--> ROLE_ADMIN
       |
       +--> READ
       +--> WRITE

user
 |
 +--> ROLE_USER
       |
       +--> READ
```

Le modèle relationnel est :

```text
users
  |
  +--> user_roles
         |
         +--> roles
                |
                +--> role_permissions
                       |
                       +--> permissions
```

Les permissions sont ajoutées au claim `authorities` du JWT afin que le backend puisse utiliser directement :

```java
@PreAuthorize("hasAuthority('READ')")
```

ou :

```java
@PreAuthorize("hasAuthority('WRITE')")
```

---

## JWT

Les jetons d'accès sont signés avec une clé privée RSA.

Exemple de JWT pour `admin` :

```json
{
  "sub": "admin",
  "aud": [
    "springstarter-api"
  ],
  "iss": "http://localhost:8081/authstarter",
  "userId": 2,
  "jti": "<identifiant-unique>",
  "authorities": [
    "READ",
    "WRITE"
  ],
  "iat": 0,
  "exp": 0
}
```

Le backend peut vérifier la signature avec la clé publique RSA et utiliser les `authorities` pour protéger ses endpoints.

---

## Refresh tokens

Contrairement aux access tokens, les refresh tokens sont des valeurs opaques générées aléatoirement.

```text
SecureRandom
     |
     v
32 octets aléatoires
     |
     v
Base64 URL-safe
     |
     v
Refresh token
```

Le refresh token brut est retourné au client mais **n'est jamais persisté**.

Seul son hash est enregistré :

```text
Refresh token
      |
      v
   SHA-256
      |
      v
64 caractères hexadécimaux
      |
      v
 PostgreSQL
```

La table `refresh_tokens` permet notamment de suivre :

```text
created_at
expires_at
used_at
revoked_at
```

Un token est valide uniquement s'il :

```text
used_at    = NULL
revoked_at = NULL
expires_at > maintenant
```

---

## Flux d'authentification

### Login

```text
POST /auth/login
      |
      v
Username / Password
      |
      v
Spring Security
      |
      +--> vérification BCrypt
      |
      v
Utilisateur authentifié
      |
      +--> chargement des permissions
      +--> génération du JWT
      +--> génération du refresh token R1
      +--> stockage de SHA-256(R1)
      |
      v
Access Token + R1
```

### Rotation du refresh token

Un refresh token ne peut être utilisé qu'une seule fois.

```text
R1 -> refresh -> R1 utilisé -> Access Token + R2

R2 -> refresh -> R2 utilisé -> Access Token + R3
```

Une tentative de réutilisation de `R1` ou `R2` retourne :

```text
HTTP 401 Unauthorized
```

### Logout

```text
POST /auth/logout
      |
      v
Refresh token
      |
      v
SHA-256
      |
      v
Recherche en base
      |
      v
revoked_at = maintenant
```

Une tentative de refresh avec ce token retourne ensuite :

```text
HTTP 401 Unauthorized
```

> La déconnexion révoque le refresh token. Un JWT déjà émis reste valide jusqu'à son expiration.

---

## API REST

L'application est disponible localement sous :

```text
http://localhost:8081/authstarter
```

### Login

```http
POST /auth/login
```

Requête :

```json
{
  "username": "user",
  "password": "password"
}
```

Réponse :

```json
{
  "accessToken": "<jwt>",
  "refreshToken": "<refresh-token>",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

Exemple :

```bash
curl -X POST \
  'http://localhost:8081/authstarter/auth/login' \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "user",
    "password": "password"
  }'
```

### Refresh

```http
POST /auth/refresh
```

Requête :

```json
{
  "refreshToken": "<refresh-token>"
}
```

Réponse :

```json
{
  "accessToken": "<nouveau-jwt>",
  "refreshToken": "<nouveau-refresh-token>",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

Les tokens inconnus, expirés, révoqués ou déjà utilisés retournent :

```text
HTTP 401 Unauthorized
```

### Logout

```http
POST /auth/logout
```

Requête :

```json
{
  "refreshToken": "<refresh-token>"
}
```

Réponse :

```text
HTTP 204 No Content
```

---

## Gestion des erreurs

Les erreurs REST sont centralisées avec un `@RestControllerAdvice` et utilisent `ProblemDetail`.

Exemple pour un refresh token invalide :

```json
{
  "type": "about:blank",
  "title": "Invalid refresh token",
  "status": 401,
  "detail": "Refresh token has expired",
  "instance": "/auth/refresh",
  "error": "invalid_refresh_token"
}
```

Un identifiant, mot de passe, access token ou refresh token ne doit jamais être exposé dans un message d'erreur ou dans les logs.

---

## Swagger UI

Les endpoints d'authentification sont documentés avec OpenAPI.

Swagger permet notamment de tester directement :

```text
POST /auth/login
POST /auth/refresh
POST /auth/logout
```

Les DTO contiennent des exemples afin que les requêtes soient préremplies lors de l'utilisation de **Try it out**.

Swagger UI est disponible localement sous :

```text
http://localhost:8081/authstarter/swagger-ui/index.html
```

---

## Base de données

L'application utilise PostgreSQL avec le schéma :

```text
authdb
```

Tables principales :

```text
users
roles
permissions
user_roles
role_permissions
refresh_tokens
```

Liquibase gère la création et l'évolution du schéma ainsi que les données de démonstration.

### Vérifier les droits

```sql
SELECT
    u.username,
    r.name AS role,
    p.name AS permission
FROM authdb.users u
JOIN authdb.user_roles ur
    ON ur.user_id = u.id
JOIN authdb.roles r
    ON r.id = ur.role_id
JOIN authdb.role_permissions rp
    ON rp.role_id = r.id
JOIN authdb.permissions p
    ON p.id = rp.permission_id
ORDER BY u.username, p.name;
```

Résultat attendu :

```text
admin   ROLE_ADMIN   READ
admin   ROLE_ADMIN   WRITE
user    ROLE_USER    READ
```

### Inspecter les refresh tokens

```sql
SELECT
    id,
    user_id,
    created_at,
    expires_at,
    used_at,
    revoked_at
FROM authdb.refresh_tokens
ORDER BY id;
```

---

## Configuration

Exemple de configuration :

```yaml
app:
  security:
    keys:
      public-key: classpath:keys/public.pem
      private-key: classpath:keys/private.pem

    jwt:
      issuer: http://localhost:8081/authstarter
      audience: springstarter-api
      access-token-duration: 1h
      refresh-token-duration: 30d
```

La clé publique doit utiliser le format :

```text
-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
```

La clé privée doit utiliser le format PKCS#8 :

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

Les clés privées de production ne doivent jamais être versionnées dans Git.

---

## Docker Compose

Afficher les services :

```bash
docker compose config --services
```

Démarrer la stack :

```bash
docker compose up -d
```

Afficher les conteneurs :

```bash
docker ps -a
```

Pour exécuter Liquibase au premier plan :

```bash
docker compose up --force-recreate liquibase
```

Afficher les logs :

```bash
docker compose logs liquibase
```

Une exécution Liquibase réussie doit se terminer avec un code de retour `0`.

---

## Développement et tests

Exécuter les tests :

```bash
mvn test
```

Compiler et vérifier le projet :

```bash
mvn clean verify
```

L'objectif est d'obtenir :

```text
Couverture JaCoCo > 80 %
```

Les tests devront principalement couvrir :

- l'authentification ;
- la génération des JWT ;
- la rotation des refresh tokens ;
- le rejet d'un token déjà utilisé ;
- l'expiration ;
- la révocation ;
- le logout ;
- la gestion des erreurs ;
- les contrôleurs REST.

---

## Intégration avec springstarter-api

L'objectif est d'utiliser `springstarter-auth` comme service d'authentification du backend.

```text
+---------------------+
|       Client        |
+----------+----------+
           |
           | username/password
           v
+---------------------+
| springstarter-auth  |
|                     |
| Authentification    |
| JWT + Refresh Token |
+----------+----------+
           |
           | JWT signé RSA
           v
+---------------------+
| springstarter-api   |
|                     |
| Resource Server     |
| Validation JWT      |
| READ / WRITE        |
+---------------------+
```

`springstarter-auth` est responsable de :

```text
authentification
utilisateurs
rôles
permissions
génération des tokens
refresh tokens
```

`springstarter-api` sera responsable de :

```text
validation de la signature JWT
validation de l'issuer
validation de l'audience
validation de l'expiration
autorisation des endpoints
logique métier
```

Le backend pourra ensuite protéger ses endpoints avec :

```java
@PreAuthorize("hasAuthority('READ')")
```

ou :

```java
@PreAuthorize("hasAuthority('WRITE')")
```

Comportement attendu :

```text
user  + READ  -> 200
user  + WRITE -> 403

admin + READ  -> 200
admin + WRITE -> 200
```

---

## Sécurité

Quelques règles importantes :

- ne jamais stocker les refresh tokens en clair ;
- ne jamais logger les mots de passe ou les tokens ;
- utiliser des access tokens de courte durée ;
- considérer les refresh tokens comme des credentials sensibles ;
- utiliser HTTPS hors environnement local ;
- ne jamais versionner les clés privées de production ;
- externaliser l'issuer, l'audience et les durées des tokens ;
- réserver les utilisateurs de démonstration aux environnements locaux et de test.

---

## Prochaines étapes

- [ ] Sécuriser les refresh simultanés
- [ ] Ajouter le nettoyage planifié des refresh tokens expirés
- [ ] Publier le projet sur GitHub
- [ ] Sécuriser la gestion du `.env` et fournir un `.env.example`
- [ ] Vérifier qu'aucun secret ou clé privée n'est présent dans Git
- [ ] Connecter `springstarter-api` à `springstarter-auth`
- [ ] Configurer le backend en OAuth2 Resource Server
- [ ] Ajouter des endpoints de démonstration `READ` et `WRITE`
- [ ] Prévoir une gestion des clés adaptée à la production