# Backend — projet tutoré ISI

Architecture microservices : `api-gateway` (Spring Cloud Gateway, réactif) qui route vers `account-micro-service` (Spring MVC).

## Prérequis

- Java 21
- Docker + Docker Compose (le plugin `docker compose`, pas l'ancien `docker-compose`)

## Lancer le projet

Les images Docker ne compilent **pas** le code : elles copient un `.jar` déjà construit. Il faut donc construire les jars avant de lancer `docker compose`.

```bash
cd account && ./mvnw clean package -DskipTests && cd ..
cd api-gateway && ./mvnw clean package -DskipTests && cd ..
docker compose up -d --build
```

`api-gateway` et `account-micro-service` ne sont **pas** exposés sur l'hôte — seul `caddy` (reverse proxy HTTPS) publie des ports (80/443) et parle aux deux services via le réseau Docker interne.

Le [Caddyfile](Caddyfile) est configuré pour le domaine `vroum.duckdns.org` : Caddy n'obtient un certificat Let's Encrypt (et ne répond correctement) que pour ce nom de domaine précis, pas pour `localhost`. Pour tester en local sans passer par le domaine, republie temporairement le port de `api-gateway` dans `docker-compose.yaml`.

En production :

```bash
curl https://vroum.duckdns.org/api/account/ping
# -> account is up
```

Arrêter :

```bash
docker compose down
```

## À chaque changement de code

Même procédure : rebuilder le jar du service modifié, puis `docker compose up -d --build`. Comme les dépendances Maven restent en cache (`~/.m2` sur l'hôte), seul le code modifié est recompilé — c'est rapide.

```bash
cd api-gateway && ./mvnw clean package -DskipTests && cd ..
docker compose up -d --build
```

### ⚠️ Piège : le cache de build Docker peut servir un vieux jar

Spring Boot fixe le mtime du jar final à une date fixe (reproductibilité des builds). Le cache de contexte de Docker/BuildKit se base sur `mtime + taille` pour détecter un changement — si la taille ne change pas franchement, il peut croire que rien n'a changé et réutiliser un **ancien** jar, même avec un code différent.

Si après un rebuild le comportement ne change pas (ou que tu vois `transferring context: quelques octets` dans les logs de build au lieu de la taille réelle du jar), force un nouveau mtime avant `docker compose up --build` :

```bash
touch api-gateway/target/*.jar
touch account/target/*.jar
docker compose up -d --build
```

## Configuration des routes de la gateway

Dans [api-gateway/src/main/resources/application.yaml](api-gateway/src/main/resources/application.yaml) :

```yaml
spring:
  cloud:
    gateway:
      server:
        webflux:
          routes:
            - id: account-micro-service
              uri: http://account-micro-service:8080   # nom du service Docker Compose
              predicates:
                - Path=/api/account/**
              filters:
                - name: StripPrefix
                  args:
                    parts: 2   # /api/account/ping -> /ping côté account
```

⚠️ Utilise la syntaxe `name`/`args` pour `StripPrefix` et non le raccourci `StripPrefix=2` — ce dernier ne compte pas correctement les segments dans cette version de Spring Cloud Gateway.

Le nom d'hôte dans `uri` doit correspondre exactement au nom du service dans `docker-compose.yaml` (ici `account-micro-service`), pas à un nom arbitraire — Docker résout les noms de service comme des hôtes DNS sur le réseau partagé.
