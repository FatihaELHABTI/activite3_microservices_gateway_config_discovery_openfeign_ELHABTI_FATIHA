# 🏗️ Architecture Microservices avec Spring Cloud

## 📋 Table des matières
- [Vue d'ensemble](#vue-densemble)
- [Technologies utilisées](#technologies-utilisées)
- [Architecture du projet](#architecture-du-projet)
- [Structure des microservices](#structure-des-microservices)
- [Prérequis](#prérequis)
- [Installation et exécution](#installation-et-exécution)
- [Endpoints API](#endpoints-api)
- [Configuration](#configuration)

## 🎯 Vue d'ensemble

Ce projet implémente une architecture microservices complète utilisant Spring Cloud pour gérer un système de facturation. L'application est composée de plusieurs services indépendants qui communiquent entre eux via Feign Client et sont orchestrés par une Gateway API et un serveur de découverte Eureka.

### Fonctionnalités principales
- Gestion des clients (Customer Service)
- Gestion des produits en inventaire (Inventory Service)
- Génération et gestion des factures (Billing Service)
- Configuration centralisée (Config Service)
- Découverte de services (Discovery Service)
- Routage intelligent via Gateway

## 🛠️ Technologies utilisées

### Backend Framework
- **Spring Boot 3.x** - Framework principal pour le développement
- **Spring Cloud** - Suite d'outils pour architectures microservices
- **Spring Data JPA** - Abstraction de la couche de persistance
- **Spring Data REST** - Exposition automatique des repositories en REST

### Communication inter-services
- **OpenFeign** - Client HTTP déclaratif pour communication synchrone
- **Spring Cloud Gateway** - Gateway API réactive avec routage dynamique

### Service Discovery & Configuration
- **Eureka Server** - Service registry pour la découverte de services
- **Spring Cloud Config Server** - Gestion centralisée des configurations

### Base de données
- **H2 Database** - Base de données en mémoire pour développement/test

### Build & Dépendances
- **Maven** - Gestion des dépendances et build
- **Lombok** - Réduction du code boilerplate

## 🏛️ Architecture du projet

```
microservices-architecture/
│
├── discovery-service/          # Eureka Server (Port 8761)
├── config-service/             # Config Server (Port 9999)
├── gateway-service/            # API Gateway (Port 8888)
├── customer-service/           # Service Clients (Port 8081)
├── inventory-service/          # Service Inventaire (Port 8082)
├── billing-service/            # Service Facturation (Port 8083)
└── config-git-repo/            # Repository Git pour configurations
```

### Flux de communication

```
Client HTTP
    ↓
Gateway Service (8888)
    ↓
Eureka Discovery (8761)
    ↓
┌─────────────┬──────────────┬──────────────┐
│             │              │              │
Customer    Inventory    Billing
Service     Service      Service
(8081)      (8082)      (8083)
                            ↓
                    Feign Clients
                    (Communication)
```

## 📦 Structure des microservices

### 1. Discovery Service (Eureka)

Service d'enregistrement et de découverte permettant aux microservices de se localiser mutuellement.

**Code principal:**
```java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(DiscoveryServiceApplication.class, args);
    }
}
```

**Configuration (application.properties):**
```properties
spring.application.name=discovery-service
server.port=8761
eureka.client.fetch-registry=false
eureka.client.register-with-eureka=false
```

### 2. Config Service

Serveur de configuration centralisée qui récupère les configurations depuis un repository Git distant.

**Code principal:**
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServiceApplication.class, args);
    }
}
```

**Configuration:**
```properties
spring.application.name=config-service
server.port=9999
spring.cloud.config.server.git.uri=https://github.com/FatihaELHABTI/config-git-repo.git
```

### 3. Gateway Service

Point d'entrée unique pour tous les microservices avec routage dynamique basé sur Eureka.

**Code principal:**
```java
@SpringBootApplication
public class GatewayServiceApplication {
    
    @Bean
    DiscoveryClientRouteDefinitionLocator discoveryClientRouteDefinitionLocator(
            ReactiveDiscoveryClient reactiveDiscoveryClient,
            DiscoveryLocatorProperties discoveryLocatorProperties) {
        return new DiscoveryClientRouteDefinitionLocator(
            reactiveDiscoveryClient, 
            discoveryLocatorProperties
        );
    }
}
```

**Accès aux services via Gateway:**
- Customer Service: `http://localhost:8888/CUSTOMER-SERVICE/api/customers`
- Inventory Service: `http://localhost:8888/INVENTORY-SERVICE/api/products`
- Billing Service: `http://localhost:8888/BILLING-SERVICE/bills/{id}`

### 4. Customer Service

Gestion des clients avec exposition automatique des endpoints REST.

**Entité Customer:**
```java
@Entity
@NoArgsConstructor @AllArgsConstructor @Getter @Setter @Builder
public class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
}
```

**Repository:**
```java
@RepositoryRestResource
public interface CustomerRepository extends JpaRepository<Customer, Long> {
}
```

**Projections disponibles:**
```java
@Projection(name = "all", types = Customer.class)
public interface CustomerProjection {
    String getName();
    String getEmail();
}
```

**Initialisation des données:**
```java
@Bean
CommandLineRunner commandLineRunner(CustomerRepository customerRepository) {
    return args -> {
        customerRepository.save(Customer.builder()
            .name("Mohamed").email("med@gmail.com").build());
        customerRepository.save(Customer.builder()
            .name("Imane").email("imane@gmail.com").build());
        customerRepository.save(Customer.builder()
            .name("Yassine").email("yassine@gmail.com").build());
    };
}
```

### 5. Inventory Service

Gestion de l'inventaire des produits.

**Entité Product:**
```java
@Entity
@NoArgsConstructor @AllArgsConstructor @Getter @Setter @Builder
public class Product {
    @Id
    private String id;
    private String name;
    private double price;
    private int quantity;
}
```

**Initialisation:**
```java
@Bean
CommandLineRunner commandLineRunner(ProductRepository productRepository) {
    return args -> {
        productRepository.save(Product.builder()
            .id(UUID.randomUUID().toString())
            .name("Computer").price(3200).quantity(11).build());
        productRepository.save(Product.builder()
            .id(UUID.randomUUID().toString())
            .name("Printer").price(1299).quantity(10).build());
        productRepository.save(Product.builder()
            .id(UUID.randomUUID().toString())
            .name("Smart Phone").price(5400).quantity(8).build());
    };
}
```

### 6. Billing Service

Service de facturation qui agrège les données de Customer et Inventory via Feign.

**Entité Bill:**
```java
@Entity
@NoArgsConstructor @AllArgsConstructor @Getter @Setter @Builder
public class Bill {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Date billingDate;
    private long customerId;
    
    @OneToMany(mappedBy = "bill")
    private List<ProductItem> productItems = new ArrayList<>();
    
    @Transient 
    private Customer customer;
}
```

**Entité ProductItem:**
```java
@Entity
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class ProductItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String productId;
    
    @ManyToOne
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private Bill bill;
    
    private int quantity;
    private double unitPrice;
    
    @Transient
    private Product product;
}
```

**Feign Clients pour communication:**
```java
@FeignClient(name = "customer-service")
public interface CustomerRestClient {
    @GetMapping("/api/customers/{id}")
    Customer getCustomerById(@PathVariable Long id);
    
    @GetMapping("/api/customers")
    PagedModel<Customer> getAllCustomers();
}

@FeignClient(name = "inventory-service")
public interface ProductRestClient {
    @GetMapping("/api/products/{id}")
    Product getProductById(@PathVariable String id);
    
    @GetMapping("/api/products")
    PagedModel<Product> getAllProducts();
}
```

**REST Controller:**
```java
@RestController
public class BillRestController {
    @Autowired private BillRepository billRepository;
    @Autowired private ProductItemRepository productItemRepository;
    @Autowired private CustomerRestClient customerRestClient;
    @Autowired private ProductRestClient productRestClient;
    
    @GetMapping("/bills/{id}")
    public Bill getBill(@PathVariable Long id) {
        Bill bill = billRepository.findById(id).get();
        bill.setCustomer(customerRestClient.getCustomerById(bill.getCustomerId()));
        bill.getProductItems().forEach(productItem -> {
            productItem.setProduct(productRestClient.getProductById(
                productItem.getProductId()
            ));
        });
        return bill;
    }
}
```

**Génération automatique de données:**
```java
@Bean
CommandLineRunner commandLineRunner(
        BillRepository billRepository,
        ProductItemRepository productItemRepository,
        CustomerRestClient customerRestClient,
        ProductRestClient productRestClient) {
    return args -> {
        Collection<Customer> customers = customerRestClient.getAllCustomers().getContent();
        Collection<Product> products = productRestClient.getAllProducts().getContent();
        
        customers.forEach(customer -> {
            Bill bill = Bill.builder()
                .billingDate(new Date())
                .customerId(customer.getId())
                .build();
            billRepository.save(bill);
            
            products.forEach(product -> {
                ProductItem productItem = ProductItem.builder()
                    .bill(bill)
                    .productId(product.getId())
                    .quantity(1 + new Random().nextInt(10))
                    .unitPrice(product.getPrice())
                    .build();
                productItemRepository.save(productItem);
            });
        });
    };
}
```

## ⚙️ Prérequis

- **JDK 17** ou supérieur
- **Maven 3.8+**
- **Git**
- **IDE** (IntelliJ IDEA, Eclipse, VS Code)

## 🚀 Installation et exécution

### Étape 1: Cloner le projet
```bash
git clone <repository-url>
cd microservices-architecture
```

### Étape 2: Ordre de démarrage des services

**Important:** Respecter cet ordre pour éviter les erreurs de dépendances.

#### 1. Démarrer Discovery Service (Eureka)
```bash
cd discovery-service
mvn spring-boot:run
```
Vérifier: http://localhost:8761

#### 2. Démarrer Config Service
```bash
cd config-service
mvn spring-boot:run
```
Vérifier: http://localhost:9999

#### 3. Démarrer Gateway Service
```bash
cd gateway-service
mvn spring-boot:run
```

#### 4. Démarrer les services métier (ordre indifférent)

**Customer Service:**
```bash
cd customer-service
mvn spring-boot:run
```

**Inventory Service:**
```bash
cd inventory-service
mvn spring-boot:run
```

**Billing Service:**
```bash
cd billing-service
mvn spring-boot:run
```

### Vérification du démarrage

Une fois tous les services démarrés, vérifier Eureka Dashboard:
```
http://localhost:8761
```

Vous devriez voir tous les services enregistrés (CUSTOMER-SERVICE, INVENTORY-SERVICE, BILLING-SERVICE, GATEWAY-SERVICE).

## 🌐 Endpoints API

### Via Gateway (recommandé)

**Customer Service:**
```bash
# Lister tous les clients
GET http://localhost:8888/CUSTOMER-SERVICE/api/customers

# Obtenir un client par ID
GET http://localhost:8888/CUSTOMER-SERVICE/api/customers/1

# Avec projection
GET http://localhost:8888/CUSTOMER-SERVICE/api/customers/1?projection=all
```

**Inventory Service:**
```bash
# Lister tous les produits
GET http://localhost:8888/INVENTORY-SERVICE/api/products

# Obtenir un produit par ID
GET http://localhost:8888/INVENTORY-SERVICE/api/products/{id}
```

**Billing Service:**
```bash
# Obtenir une facture avec détails complets
GET http://localhost:8888/BILLING-SERVICE/bills/1
```

### Accès direct (développement)

**Customer Service:**
```bash
GET http://localhost:8081/api/customers
```

**Inventory Service:**
```bash
GET http://localhost:8082/api/products
```

**Billing Service:**
```bash
GET http://localhost:8083/bills/1
```

### Consoles H2

- Customer DB: http://localhost:8081/h2-console
- Inventory DB: http://localhost:8082/h2-console
- Billing DB: http://localhost:8083/h2-console

**Paramètres de connexion:**
- JDBC URL: `jdbc:h2:mem:{service-name}-db`
- Username: `sa`
- Password: *(vide)*

## 🔧 Configuration

### Configuration centralisée (Git Repository)

Les configurations sont stockées dans le repository Git et suivent la convention de nommage:
```
{application-name}-{profile}.properties
```

**Exemples:**
- `customer-service.properties` - Configuration par défaut
- `customer-service-dev.properties` - Configuration développement
- `customer-service-prod.properties` - Configuration production

### Profils disponibles

Changer de profil en ajoutant à la ligne de commande:
```bash
mvn spring-boot:run -Dspring-boot.run.arguments=--spring.profiles.active=dev
```

Ou dans `application.properties`:
```properties
spring.profiles.active=dev
```

### Structure des fichiers de configuration

**application.properties (global):**
```properties
global.params.p1=555
global.params.p2=777
spring.h2.console.enabled=true
spring.cloud.discovery.enabled=true
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
eureka.instance.prefer-ip-address=true
management.endpoints.web.exposure.include=*
```

**customer-service.properties:**
```properties
customer.params.x=11
customer.params.y=22
spring.datasource.url=jdbc:h2:mem:customers-db
spring.data.rest.base-path=/api
```

## 📊 Schéma de la base de données

### Customer Service
```
CUSTOMER
├── id (PK)
├── name
└── email
```

### Inventory Service
```
PRODUCT
├── id (PK)
├── name
├── price
└── quantity
```

### Billing Service
```
BILL                          PRODUCT_ITEM
├── id (PK)                   ├── id (PK)
├── billing_date              ├── product_id
└── customer_id               ├── bill_id (FK)
                              ├── quantity
                              └── unit_price
```

## 🐛 Troubleshooting

### Problème: Service non enregistré dans Eureka
**Solution:** Vérifier que Eureka est démarré et accessible sur le port 8761.

### Problème: Erreur de configuration
**Solution:** Vérifier que Config Service est démarré et que le repository Git est accessible.

### Problème: Feign Client timeout
**Solution:** Augmenter le timeout dans `application.properties`:
```properties
feign.client.config.default.connectTimeout=5000
feign.client.config.default.readTimeout=5000
```

### Problème: Port déjà utilisé
**Solution:** Changer le port dans `application.properties` ou arrêter l'application utilisant le port.

## 🎓 Concepts clés

### Service Discovery (Eureka)
Permet aux microservices de s'enregistrer et de se découvrir dynamiquement sans configuration statique.

### Config Server
Centralise la gestion des configurations dans un repository Git, permettant le rafraîchissement dynamique sans redémarrage.

### API Gateway
Fournit un point d'entrée unique, gère le routage, l'équilibrage de charge et peut implémenter des fonctionnalités transversales (sécurité, monitoring).

### Feign Client
Simplifie la communication inter-services avec une approche déclarative basée sur des interfaces.

### Spring Data REST
Expose automatiquement les repositories JPA en tant qu'endpoints REST avec HATEOAS.

## 📝 Améliorations possibles

- Implémenter la sécurité avec OAuth2/JWT
- Ajouter un circuit breaker (Resilience4j)
- Implémenter le tracing distribué (Sleuth + Zipkin)
- Ajouter la gestion des logs centralisée (ELK Stack)
- Containeriser avec Docker et Docker Compose
- Implémenter des tests d'intégration
- Ajouter un service de cache (Redis)
- Implémenter des événements asynchrones (Kafka/RabbitMQ)

## 👥 Auteur

**Fatiha ELHABTI**

## 📄 Licence

Ce projet est sous licence MIT.
