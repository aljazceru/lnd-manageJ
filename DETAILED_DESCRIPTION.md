# lnd-manageJ Detailed Description

lnd-manageJ is a modular Java service that monitors an existing `lnd` node, stores historical data, and exposes a REST API and web interface for inspection. The project is organized as many Gradle modules, each handling a domain such as balances, forwarding history, private channels, transactions, and UI.

## Architecture Overview

- **Entry point** – `Application.java` in the `application` module configures Spring Boot, enables scheduling of background tasks, and sets up Flyway for database migrations.
- **Configuration** – Settings are loaded from `${user.home}/.config/lnd-manageJ.conf` as defined in `application.properties`.
- **Modules** – `settings.gradle.kts` lists submodules like `balances`, `payments`, `onlinepeers`, `forwarding-history`, `transactions`, `pickhardt-payments`, `web`, and `ui`.
- **Persistence** – Each module includes JPA entities for its tables. Flyway migrations create tables such as `balances`, `forwarding_events`, `online_peers`, `payments`, `settled_invoices`, and `transactions` (see `HTWO1_0_3__base.sql`).
- **Backend services** – The `backend` module orchestrates communication with `lnd` via gRPC (wrapped in the `grpc-adapter` and `grpc-client` modules) and caches responses using Caffeine.
- **Scheduling** – Collectors like `BalancesUpdater`, `ForwardingHistory`, and `OnlinePeersUpdater` are annotated with `@Scheduled` and run continuously to record data into the database.
- **REST layer** – Controllers in the `web` module provide endpoints under `/api`, for example `/api/channel/{ID}/balance` and `/api/node/{PUBKEY}/details`.
- **UI** – The `ui` module uses Thymeleaf templates to render the same information for browser users. A demo can be launched with `./ui-demo.sh`.

## Data Collection and Storage

- **Balances** – Recorded by `BalancesUpdater` and stored in the `balances` table keyed by channel ID and timestamp.
- **Forwarding history** – `ForwardingHistory` fetches forwarding events and inserts them into `forwarding_events`.
- **Payments and Invoices** – Modules `payments` and `invoices` poll gRPC every few seconds. Combined data identifies self-payments, which are stored in `self_payments`.
- **Online status** – `OnlinePeersUpdater` records whether each peer is online in `online_peers` every five minutes.
- **Transactions** – On-chain transaction details are retrieved from public APIs (Bitaps and Blockcypher) and stored in `transactions`.
- **Fee rates and private channels** – Separate modules track fee policy changes and which channels are private.

## REST API

The README lists numerous endpoints. Examples:

- `GET /api/status/open-channels` – list of open channel IDs.
- `GET /api/channel/{ID}/balance` – local and remote balances, including reserves.
- `GET /api/channel/{ID}/warnings` – warnings such as significant balance fluctuation.
- `GET /api/node/{PUBKEY}/warnings` – node-level warnings like poor uptime or low rating.
- `GET /api/warnings` – aggregated warnings for all nodes and channels.

Channels IDs can be supplied in various formats including short IDs (`123456:123:1`), compact format (`123456x123x1`), 64‑bit numeric ID, or channel point (funding transaction with output index).

## Configuration Options

`example-lnd-manageJ.conf` documents user-adjustable parameters:

```ini
[warnings]
channel_fluctuation_lower_threshold=10
channel_fluctuation_upper_threshold=90
online_percentage_threshold=80
online_changes_threshold=50
node_rating_threshold=1000
```

Pickhardt payments are configurable under `[pickhardt-payments]` with settings such as `quantization`, `piecewise_linear_approximations`, and `max-cltv-expiry`.

Ratings settings in `[ratings]` specify `minimum_age_in_days` and `days_for_analysis`.

## Ratings

`rating.md` explains how channel and node ratings accumulate metrics from received payments, forwarding fees, rebalancing activity, and potential earnings. The value is normalized by analysis days and average balance. Nodes with a rating below the threshold (default 1,000) generate warnings.

## Database Schema

`HTWO1_0_3__base.sql` shows the initial schema. A snippet includes:

```sql
create table if not exists balances (
    channel_id      bigint not null,
    timestamp       bigint not null,
    local_balance   bigint not null,
    local_reserved  bigint not null,
    remote_balance  bigint not null,
    remote_reserved bigint not null,
    primary key (channel_id, timestamp)
);

create table if not exists forwarding_events (
    event_index      integer not null primary key,
    amount_incoming  bigint  not null,
    amount_outgoing  bigint  not null,
    channel_incoming bigint  not null,
    channel_outgoing bigint  not null,
    timestamp        bigint  not null
);
```

Other tables include `payments`, `payment_routes`, `settled_invoices`, and `transactions` with indexes for efficient queries.

## Application Properties

`application.properties` configures the service:

```properties
server.address=127.0.0.1
server.port=8081
spring.datasource.url=jdbc:postgresql://localhost:5432/lndmanagej
spring.datasource.username=bitcoin
spring.flyway.baseline-on-migrate=true
resilience4j.ratelimiter.instances.bitaps.limit-for-period=1
```

The configuration file path is `lndmanagej.configuration-path=${user.home}/.config/lnd-manageJ.conf` so updates can be applied at runtime.

## Caching and Performance

Most services cache their results. Node aliases and channel lists are cached for several minutes, whereas balance information is cached for under a second. The README warns that some data may be outdated and that first-time requests can be slow due to transaction downloads.

## Scheduling

Collectors run periodically via Spring's scheduler. Example intervals:

- Payments and invoice collectors poll every 10 seconds.
- Online peer status is recorded every 5 minutes.
- Balances and forwarding events are updated in similar intervals.

## Pickhardt Payments

The `pickhardt-payments` module implements advanced path finding and splitting based on Pickhardt's algorithm. Classes such as `MinCostFlowSolver`, `MultiPathPaymentSplitter`, and `MultiPathPaymentSender` attempt to send payments using multiple paths while minimizing cost. The behavior is controlled by the `[pickhardt-payments]` section in the configuration file.

## Running the Service

The README describes running with H2 or PostgreSQL. Docker Compose can start the service with PostgreSQL, exposing port 8081. When running in Docker, add `host.docker.internal` to reach `lnd` on the host. The UI demo can be started with `./ui-demo.sh`.

## Privacy Notice

Downloading transaction details from Bitaps and Blockcypher leaks channel information, as noted in the README.

---

This description summarizes the repository's architecture, data flow, and configuration. It can serve as a specification for re‑implementing the same functionality in another language.
