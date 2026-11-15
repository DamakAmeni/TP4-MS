# TP4 Spring Cloud : Microservices « Calculatrice » avec Eureka, Gateway et OpenFeign

- **Matière** : Architecture MicroServices
- **Filière** : MPSRCC1, Faculté des Sciences de Sfax
- **Enseignantes** : Corinne Amel Zayani, Amal Abid
- **Auteur** : Damak Ameni
- **Année universitaire** : 2026-2027

## 1. Présentation

Ce projet reprend les deux microservices du TP3 (`calcul-service` et `historique-service`) et leur ajoute une architecture Spring Cloud :

- un **annuaire de services** (Eureka Server) ;
- un **client REST déclaratif** (OpenFeign) avec répartition de charge (Spring Cloud LoadBalancer) ;
- une **passerelle** (Spring Cloud Gateway) à routes statiques (YAML et Java) puis dynamiques ;
- une documentation et des tests par **Swagger UI** (springdoc-openapi) ;
- **Lombok** pour générer constructeurs, getters et setters.

Versions : Spring Boot 4.0.9, Spring Cloud 2025.1.4, springdoc-openapi 3.0.3, Java 17.

## 2. Architecture

| Programme | Port | Rôle |
|---|---|---|
| eureka-service | 8761 | Annuaire des services |
| calcul-service (instance 1) | 8081 | Quatre calculs |
| calcul-service (instance 2) | 8083 | Second exemplaire du même programme |
| historique-service | 8082 | Enregistre les opérations, appelle calcul-service |
| gateway-service | 8888 | Point d'entrée unique des clients |

## 3. Structure du projet

```
calculatrice-spring-cloud/
|-- pom.xml                 (pom parent, packaging pom)
|-- eureka-service/
|-- calcul-service/
|-- historique-service/
|-- gateway-service/
`-- README.md
```

## 4. Lancement

Ordre : eureka-service, calcul-service (8081), calcul-service (8083), historique-service, gateway-service.

Sans IntelliJ, à la racine du projet :

```
mvn clean package -DskipTests
java -jar eureka-service/target/eureka-service-0.0.1-SNAPSHOT.jar
java -jar calcul-service/target/calcul-service-0.0.1-SNAPSHOT.jar
java -jar calcul-service/target/calcul-service-0.0.1-SNAPSHOT.jar --server.port=8083
java -jar historique-service/target/historique-service-0.0.1-SNAPSHOT.jar
java -jar gateway-service/target/gateway-service-0.0.1-SNAPSHOT.jar
```

Adresses utiles :

- Annuaire Eureka : http://localhost:8761
- Swagger UI de la passerelle : http://localhost:8888/swagger-ui.html
- Routes de la passerelle : http://localhost:8888/actuator/gateway/routes

## 5. Résultats des tests

### D.4 : historique-service (directement, port 8082)

| N° | Requête | Corps | Statut | Réponse |
|---|---|---|---|---|
| 1 | POST /operations | `{"operateur":"somme","a":8,"b":2}` | 201 | id 1, résultat 10, en-tête `Location: http://localhost:8082/operations/1` |
| 2 | POST /operations | `{"operateur":"produit","a":6,"b":7}` | 201 | id 2, résultat 42 |
| 3 | POST /operations | `{"operateur":"division","a":7,"b":2}` | 201 | id 3, résultat 3.5 |
| 4 | POST /operations | `{"operateur":"division","a":7,"b":0}` | 400 | `{"erreur":"Erreur : division par 0 impossible"}` |
| 5 | POST /operations | `{"operateur":"racine","a":9,"b":0}` | 400 | `{"erreur":"Operateur inconnu : racine"}` |
| 6 | GET /operations | aucun | 200 | tableau des opérations 1, 2 et 3 |
| 7 | GET /operations/2 | aucun | 200 | opération 2 (produit, 6 et 7, résultat 42) |

Les traces des requêtes 1, 2 et 3 alternent entre `[calcul-service:8081]` et `[calcul-service:8083]`.

### E.2 : routes statiques en YAML (adresses fixes)

| N° | Requête (préfixe http://localhost:8888) | Statut | Réponse | Instance |
|---|---|---|---|---|
| 1 | GET /calcul/somme?a=2&b=3 | 200 | 5 | toujours 8081 (adresse fixe) |
| 2 | GET /calcul/division?a=7&b=0 | 400 | Erreur : division par 0 impossible | 8081 |
| 3 | GET /operations | 200 | tableau JSON des opérations | historique-service |
| 4 | POST /operations (soustraction, 10 et 4) | 201 | résultat 6, `Location: http://localhost:8888/operations/…` | historique-service, puis calcul-service |
| 5 | GET /calcul/division/detail?a=2&b=3 | 200 | `{"operation":"division","a":2,"b":3,"resultat":0.666…}` | calcul-service |
| 6 | GET /bonjour (navigateur) | 404 | erreur de la passerelle | la passerelle (aucune route) |

### E.3 : routes `lb://`

Après passage à `lb://CALCUL-SERVICE` et `lb://HISTORIQUE-SERVICE`, six envois de `GET /calcul/somme?a=2&b=3` sont répartis à tour de rôle entre les instances 8081 et 8083.

### E.4 : routes statiques en Java

Les mêmes six requêtes donnent les mêmes résultats avec le bean `RouteLocator`.

### E.5 : routes dynamiques

| Requête (préfixe http://localhost:8888) | Statut | Réponse |
|---|---|---|
| GET /calcul-service/calcul/somme?a=2&b=3 | 200 | 5 |
| GET /calcul-service/calcul/produit/6/7 | 200 | 42 |
| GET /historique-service/operations | 200 | tableau JSON des opérations |
| GET /calcul/somme?a=2&b=3 | 404 | plus de route `/calcul/**` |
| GET /actuator/gateway/routes | 200 | une route par service de l'annuaire (CALCUL-SERVICE, HISTORIQUE-SERVICE, GATEWAY-SERVICE) |

Quand `historique-service` est arrêté, sa route disparaît de `/actuator/gateway/routes` ; elle réapparaît dès son redémarrage.

## 6. Réponses aux questions

### Partie A : questions de conception

**1.** Le nom est associé à ses adresses par l'annuaire Eureka Server. La correspondance est mise à jour à chaque inscription d'une instance au démarrage, à chaque désinscription à l'arrêt et par les battements de cœur : une instance silencieuse est retirée après un délai. Les clients (historique-service, passerelle) en gardent une copie locale, relue périodiquement.

**2.** Avantage : le client ne connaît qu'une adresse (8888), et la passerelle centralise le routage, la sécurité et la journalisation. Inconvénient : elle est un point de défaillance unique et ajoute un saut réseau, donc de la latence.

**3.** Avec la stratégie à tour de rôle, chaque instance reçoit 3 requêtes sur 6. Le client ne peut pas s'en apercevoir : l'URI et la réponse sont identiques ; seules les traces dans les consoles le montrent.

### Partie B

**Question B.1.** Chaque programme est un processus distinct (une JVM et un Tomcat à lui) qui écoute sur un port différent : 8761, 8081, 8082, 8083, 8888. Il n'y a de conflit que si deux processus veulent le même port.

### Partie C

**Question C.1.** Le nom (`CALCUL-SERVICE`) désigne le service logique, commun à toutes ses instances : c'est lui qu'un appelant utilise (`@FeignClient`, `lb://`). L'identifiant (`calcul-service:8081`, `calcul-service:8083`) distingue chaque instance dans l'annuaire, pour son inscription, ses battements de cœur et sa désinscription. Le répartiteur choisit parmi les instances qui portent le même nom.

**Question C.2.** Sans `--server.port=8083`, la seconde instance lirait `server.port=8081` dans `application.properties`. Ce port étant déjà occupé par la première instance, le démarrage échouerait (port déjà utilisé).

### Partie D

**Question D.1.** Les six traces alternent entre `[calcul-service:8081]` et `[calcul-service:8083]`. [Complétez avec l'ordre observé, par exemple : 8081, 8083, 8081, 8083, 8081, 8083.] La règle est de prendre à chaque appel l'instance suivante de la liste, puis de recommencer au début. Cette stratégie s'appelle le tour de rôle (round robin).

**Question D.2.** Avec `name = "CALCULSERVIC"`, j'obtiens un statut 503 avec le corps `{"erreur":"calcul-service injoignable"}`. La réponse est produite par le second bloc `catch` (`FeignException`) : le nom ne correspond à aucune instance de l'annuaire, donc LoadBalancer répond 503, qui n'est pas une erreur 400 (`FeignException.BadRequest`). Le bon nom a ensuite été remis.

### Partie E

**Question E.1.** La passerelle ajoute à la requête les en-têtes `X-Forwarded-Host`, `X-Forwarded-Port` et `X-Forwarded-Proto` (et `X-Forwarded-Prefix` en routes dynamiques). Grâce à `server.forward-headers-strategy=framework`, historique-service les utilise pour reconstruire l'adresse vue par le client : `ServletUriComponentsBuilder.fromCurrentRequest()` produit donc `http://localhost:8888/operations/…` et non le port 8082.

**Question E.2.** En E.3 l'URI est `/calcul/somme` ; en E.5 elle est `/calcul-service/calcul/somme`. La forme E.3 donne des URI propres, qui ne révèlent pas les noms des services, et chaque route est contrôlée explicitement. La forme E.5 n'exige aucune modification quand on ajoute un microservice, car les routes suivent l'annuaire. C'est donc E.3 qui oblige à modifier la passerelle.

**Question E.3.** La passerelle est elle-même enregistrée dans Eureka sous `GATEWAY-SERVICE`, et le locator crée une route pour chaque service de l'annuaire, elle-même comprise. Ce n'est pas bloquant, mais cela expose ses propres URL (par exemple son actuator) sous `/gateway-service/**`. On peut l'exclure du locator ou la retirer de l'annuaire.

## 7. Questions de synthèse

**1. Où observer chaque différence du tableau A.3**

| Aspect | Étape |
|---|---|
| Nombre de programmes | B (annuaire), C.5 (2e instance), E (passerelle) |
| Dépendances Maven | B.2, C.1, C.7, E.1 |
| Adresse d'un microservice | C.7 (suppression de `calcul.service.url`), D.2 |
| Appel entre microservices | D.1 à D.3 |
| Erreurs d'appel | D.3, D.4 (tests 4 et 5), D.2 |
| Point d'entrée du client | E.2 |
| Plusieurs instances | C.5, D.4, E.3 |
| Serveur embarqué | B.5 et C.4 (Tomcat), E.2 (Netty) |
| Arrêt d'un microservice | C.6 |
| Organisation du code | B.0 |
| Outil de test | C.4, D.4, E.2 (Swagger UI) |
| Classes de données | C.1 bis, C.7 |

**2. Trajet de `POST :8888/historique-service/operations`**

1. Le client envoie `POST http://localhost:8888/historique-service/operations` à la passerelle (requête HTTP n°1).
2. La passerelle fait correspondre le prédicat `Path=/historique-service/**`, consulte l'annuaire (sa copie locale) pour les instances de `HISTORIQUE-SERVICE`, en choisit une, retire le préfixe et transmet `POST /operations` (requête HTTP n°2).
3. `OperationController.creer` appelle `calculClient.calculer("somme", 1, 2)`. OpenFeign consulte l'annuaire pour `CALCUL-SERVICE`, LoadBalancer choisit 8081 ou 8083 et la requête `GET /calcul/somme?a=1&b=2` est envoyée (requête HTTP n°3).
4. calcul-service répond 200 avec `3`.
5. historique-service enregistre l'opération et répond 201 avec l'en-tête `Location` et le JSON de l'opération.
6. La réponse remonte : calcul-service, historique-service, passerelle, client.

**3. Répartiteurs de charge.** Le répartiteur est fourni avec le client Eureka, donc présent dans les trois programmes. Il n'est utilisé que dans deux : historique-service (il résout `CALCUL-SERVICE`) et gateway-service (il résout `CALCUL-SERVICE` et `HISTORIQUE-SERVICE`, plus `GATEWAY-SERVICE` en routes dynamiques). calcul-service n'appelle personne et n'en résout aucun.

**4. Dépendances nécessaires**

- Client Eureka : dans les trois. calcul-service s'inscrit pour être trouvé ; historique-service et la passerelle s'inscrivent et lisent aussi l'annuaire.
- OpenFeign : seulement dans historique-service, le seul à appeler un autre microservice.
- Spring Cloud Gateway : seulement dans gateway-service, qui est la passerelle.

**5. Adresse écrite en dur et répartition de charge.** Avec `calcul.service.url=http://localhost:8081`, historique-service n'appelle qu'une seule instance, même si trois exemplaires tournent : aucune répartition de charge. Si l'instance change de port ou de machine, il faut modifier la configuration et relancer le client. Dans ce TP, Eureka tient la liste des instances par nom, le client Eureka les inscrit automatiquement, Spring Cloud LoadBalancer choisit une instance à tour de rôle, OpenFeign appelle le service par son nom, et la passerelle (`lb://`) fait de même pour les clients. Si une instance tombe, les autres prennent le relais, et le 503 n'apparaît que s'il n'en reste aucune.

**6. Deux instances d'historique-service (8082 et 8084).** Chaque instance garde ses opérations en mémoire, et les POST sont répartis entre elles. `GET /historique-service/operations` renvoie donc la liste de l'instance qui répond, différente d'un appel à l'autre, avec des `id` qui peuvent se répéter. Le principe mis en défaut est celui des services sans état (stateless), qui permettent la mise à l'échelle horizontale. On le corrigerait en stockant les opérations dans une base de données partagée (MySQL ou PostgreSQL avec Spring Data JPA) ou dans un cache distribué comme Redis.

**7. Construire et lancer sans IntelliJ.** Voir la section 4 : `mvn clean package -DskipTests` à la racine construit les quatre modules, puis cinq commandes `java -jar` dans cinq terminaux, dans l'ordre indiqué.

**8. `@GetMapping("/calcul/{operateur}")`.** Dans `CalculController`, l'annotation décrit une requête reçue : elle associe l'URI à la méthode qui la traite (côté serveur). Dans `CalculClient`, elle décrit une requête à envoyer : OpenFeign construit à partir d'elle `GET /calcul/{operateur}?a=..&b=..` vers une instance du service (côté client). C'est la même annotation, lue dans l'autre sens.

**9. Indépendance des modules.** Le regroupement ne remet pas en cause l'indépendance. Chaque microservice a sa propre JVM (processus séparé), son port, son .jar exécutable (son propre `pom.xml`) et son cycle de vie : on le lance, l'arrête ou le redéploie sans toucher aux autres (arrêter l'instance 8083 n'a pas arrêté les autres). Le projet commun ne sert qu'à organiser le code et à construire ensemble : une seule fenêtre IntelliJ, une seule commande Maven.

**10. Adresse de la zone Servers.** Elle vient des en-têtes `X-Forwarded-*` ajoutés par la passerelle : Host, Port et Proto en E.2, plus `X-Forwarded-Prefix: /calcul-service` en E.5. Grâce à `server.forward-headers-strategy=framework`, le microservice reconstruit l'adresse vue par le client, et springdoc l'écrit dans la description OpenAPI. Sans cette propriété, le microservice ignore les en-têtes et annonce sa propre adresse (`http://localhost:8081`). Swagger UI enverrait alors Execute directement au microservice depuis une page servie par le port 8888 : le navigateur bloquerait la requête (CORS, « Failed to fetch ») et la passerelle serait contournée.

**11. Lombok.** Les constructeurs, getters et setters ne sont pas dans `Operation.java` mais dans `Operation.class`, dans `target/classes` : le processeur d'annotations de Lombok les ajoute au bytecode pendant la compilation. Lombok peut être exclu du .jar exécutable parce qu'il ne sert qu'à compiler : à l'exécution, le bytecode contient déjà toutes les méthodes générées.

## 8. Difficultés rencontrées

- **Variante de la passerelle** : l'assistant Spring Boot proposait `gateway-server-webmvc` ; il faut `spring-cloud-starter-gateway-server-webflux`, sinon la propriété `spring.cloud.gateway.server.webflux.routes` est ignorée.
- **JSON avec champs absents** : avec Spring Boot 4 (Jackson 3), `{"operateur":"somme","a":8,"b":2}` était refusé (400) car `id` et `resultat` sont des types primitifs. Correction : `spring.jackson.deserialization.fail-on-null-for-primitives=false` dans historique-service.
- **Swagger UI derrière la passerelle** : la description OpenAPI annonçait l'adresse du microservice (8081), d'où « Failed to fetch » (CORS). Correction en configuration statique : filtre `PreserveHostHeader`.
- **Adresse IP d'instance** : à un changement de réseau, Eureka a enregistré une adresse 169.254.x.x injoignable (erreur 500 de la passerelle). Correction : `eureka.instance.ip-address=127.0.0.1`.
- **Délai de rafraîchissement** : juste après l'arrêt d'une instance, la passerelle peut encore l'appeler (500, « Connection refused ») jusqu'au prochain rafraîchissement de l'annuaire (5 s avec les réglages du TP).
- **Zone Servers en routes dynamiques** : malgré le filtre `SetRequestHeader X-Forwarded-Prefix` sur les routes, la zone Servers n'affichait pas le préfixe `/calcul-service`. Un test direct avec les en-têtes `Host` et `X-Forwarded-Prefix` donne bien `http://localhost:8888/calcul-service` : le microservice sait l'exploiter, mais je n'ai pas identifié pourquoi l'en-tête n'arrivait pas par la passerelle.