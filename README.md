# Turistička aplikacija – SOA (tim 13)

Projekat iz predmeta **Servisno orijentisane arhitekture (SOA)** – turistička aplikacija zasnovana na
mikroservisima.

## O projektu

Aplikacija omogućava korisnicima da pregledaju i kupuju ture, pišu blogove o putovanjima, prate druge korisnike
i ostavljaju komentare i lajkove. Sistem je podeljen na nezavisne servise koji komuniciraju preko REST-a, gRPC-a i
reda poruka, a svi zahtevi klijenta prolaze kroz zajednički API gateway.

## Arhitektura

| Servis | Namena | Baza |
|---|---|---|
| `Gateway` | ulazna tačka, rutiranje zahteva, REST → gRPC (grpc-gateway) | – |
| `Stakeholders` | korisnički nalozi, profili, autentifikacija | PostgreSQL |
| `Blog` | blogovi, slike, komentari, lajkovi | MongoDB |
| `Followers` | praćenje korisnika | Neo4j |
| `Tours` | ture, ključne tačke, recenzije tura (C# / .NET 8) | PostgreSQL |
| `Shopping` | korpa i kupovina (Go) | PostgreSQL |

Pored servisa, u repozitorijumu su i:
- `monitoring` – centralizovano logovanje i praćenje zahteva (Fluent Bit, Loki, Grafana, Jaeger),
- `docker-compose.yml` – pokretanje celog sistema (uključujući NATS za razmenu poruka),
- `protoc` / `proto` – definicije gRPC servisa.

## Tehnologije

- Go (mikroservisi), Docker i Docker Compose
- PostgreSQL, MongoDB, Neo4j
- gRPC i grpc-gateway, NATS
- OpenTelemetry i Jaeger (distribuirano praćenje)
- Fluent Bit, Loki i Grafana (logovanje)

## Pokretanje

```
docker compose up --build
```

Monitoring stek se pokreće posebno:

```
cd monitoring
docker compose up
```

## Moj doprinos

### 1. Blog servis
- Kreiranje i dobavljanje blogova iz **MongoDB**-a (`BlogHandler`, `BlogService`, `BlogRepository`, model `Blog`).
- Kreiranje bloga sa slikom: otpremanje fajla, čuvanje na serveru i prosleđivanje preko gateway-a;
  integracija sa frontend delom.
- **Ostavljanje komentara** na blogove (`AddComment`), uz proveru identiteta autora preko `X-Account-Id` zaglavlja
  i dobavljanje podataka o korisniku iz Stakeholders servisa (`StakeholderService`).
- **Filtriranje blogova prema praćenim korisnicima:** korisnik vidi sopstvene blogove i blogove korisnika koje prati
  (`FindAllBlogs`, `FollowerService`).

### 2. Followers servis
- Novi servis `Followers` (handler, servis, repozitorijum, DTO objekti, Dockerfile) koji čuva odnose praćenja
  u **Neo4j** grafovskoj bazi.
- Endpointi za praćenje korisnika (`FollowUser`) i proveru da li neko prati nekoga (`IsFollowing`).
- Komunikacija sa Stakeholders servisom radi dobavljanja profila (`StakeholdersService`).
- Dodavanje servisa u `docker-compose.yml` zajedno sa Neo4j bazom.

### 3. gRPC
- gRPC komunikacija za kreiranje i dobavljanje blogova (`blog.proto`, `BlogGrpcHandler`):
  metode `CreateBlog` i `GetBlog`.
- Konfiguracija gateway-a (grpc-gateway) tako da se REST zahtevi klijenta prevode u gRPC pozive ka Blog servisu.

### 4. Centralizovano logovanje
- Stek **Fluent Bit → Loki → Grafana** (`monitoring/`): logovi kontejnera se prikupljaju (Fluent Bit forward input),
  šalju u Loki i prikazuju u Grafana-i (provisioning izvora podataka).
- U glavnom `docker-compose.yml` definisan je zajednički `fluentd` log driver (`x-logging`) koji šalje logove
  servisa (gateway, stakeholders, blog, tours, shopping, NATS i baze) na Fluent Bit (port 24224).

### 5. Distribuirano praćenje (tracing)
- Integracija **OpenTelemetry**-ja u servise Blog i Stakeholders (`opentelemetry.go`) sa izvozom u **Jaeger**
  (uz mogućnost izvoza u fajl).
- Propagacija trace konteksta kroz HTTP i gRPC pozive između servisa.
- Jaeger kao izvor podataka u Grafana-i i u monitoring `docker-compose.yml`.
