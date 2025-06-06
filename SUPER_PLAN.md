# Reverse Engineering Plan: lnd-manageJ

This document collects a very detailed specification of all notable features in the repository. It is intended as a blueprint for re-implementing the application in another language.

## Overall Purpose
- Continuously monitor a running `lnd` node.
- Store historical data about balances, forwarding events, invoices, payments, on-chain transactions and channel status.
- Provide REST endpoints and a simple HTML UI to inspect the gathered data.
- Offer advanced payment functionality via "Pickhardt Payments" (multi-path payments with liquidity-aware routing).

## Module Layout
- `application` – Spring Boot entrypoint and global configuration (port, datasource, Flyway, etc.).
- `backend` – gRPC wrappers and caching for lnd communication.
- `model` – domain objects like `ChannelId`, `Coins`, `ForwardingEvent`, `FlowReport`, etc. (see below for a full list).
- `balances`, `onlinepeers`, `forwarding-history`, `transactions`, `payments`, `invoices`, `selfpayments` – each contains persistence + background updaters for a certain data type.
- `web` – REST controllers returning DTOs.
- `ui` – Thymeleaf based web UI (controllers + templates).
- `pickhardt-payments` – path finding and payment splitting/sending logic.

## REST Endpoints
All endpoints are reachable under `http://localhost:8081/`.

### StatusController (`/api/status/`)
- `/synced-to-chain` → boolean, direct from `lnd.getinfo`.
- `/block-height` → current block height from `lnd.getinfo`.
- `/open-channels` → JSON list of channel IDs for channels currently open.
- `/open-channels/pubkeys` → JSON list of pubkeys with at least one open channel.
- `/all-channels` → JSON list of channel IDs for all local channels (open/closed/etc.).
- `/all-channels/pubkeys` → JSON list of pubkeys appearing in any local channel.
- `/known-channels` → total number of channels visible in the network graph.
- `/nodes-with-high-incoming-fee-rate` → list of pubkeys with a high inbound fee rate.

### ChannelController (`/api/channel/{channelId}`)
- `/` → `ChannelDto` with open height, remote pubkey, capacity, status …
- `/details` → aggregates the full `ChannelDetailsDto` (balances, fees, flow, rating, warnings, on‑chain costs, etc.).
- `/balance` → local/remote balance and reserves.
- `/policies` → local + remote fee settings (base fee, ppm, inbound parameters, enabled flag).
- `/close-details` → if closed: initiator, close height, force/breach flags.
- `/fee-report` → forwarding fees earned and sourced for this channel.

### FlowController
- `/api/channel/{channelId}/flow-report` → `FlowReportDto` for the full history.
- `/api/channel/{channelId}/flow-report/last-days/{N}` → same but limited to recent `N` days.
- `/api/node/{pubkey}/flow-report` → aggregated over all channels with that peer.
- `/api/node/{pubkey}/flow-report/last-days/{N}` → as above with time limit.

### OnChainCostsController
- `/api/node/{pubkey}/on-chain-costs` → aggregated open/close/sweep costs.
- `/api/channel/{channelId}/open-costs` → on-chain fees paid to open the channel.
- `/api/channel/{channelId}/close-costs` → on-chain fees paid to close.
- `/api/channel/{channelId}/sweep-costs` → costs to sweep funds after force close.

### NodeController (`/api/node/{pubkey}`)
- `/alias` → alias as plain string.
- `/details` → `NodeDetailsDto` (channels, rating, warnings, etc.).
- `/open-channels` → channel IDs for currently open channels with that peer.
- `/all-channels` → all historical channel IDs with that peer.
- `/balance` → aggregated `BalanceInformationDto` for all channels with the peer.
- `/fee-report` → aggregated fees earned/sourced from this peer.
- `/fee-report/last-days/{N}` → same but for recent `N` days.

### RebalancesController
- `/api/node/{pubkey}/rebalance-source-costs`
- `/api/node/{pubkey}/rebalance-source-amount`
- `/api/channel/{channelId}/rebalance-source-costs`
- `/api/channel/{channelId}/rebalance-source-amount`
- `/api/node/{pubkey}/rebalance-target-costs`
- `/api/node/{pubkey}/rebalance-target-amount`
- `/api/channel/{channelId}/rebalance-target-costs`
- `/api/channel/{channelId}/rebalance-target-amount`
- `/api/channel/{channelId}/rebalance-support-as-source-amount`
- `/api/node/{pubkey}/rebalance-support-as-source-amount`
- `/api/channel/{channelId}/rebalance-support-as-target-amount`
- `/api/node/{pubkey}/rebalance-support-as-target-amount`
(All endpoints return milli-satoshis as long values.)

### SelfPaymentsController
- `/api/channel/{channelId}/self-payments-from-channel` → list of self-payments sending funds out of this channel.
- `/api/node/{pubkey}/self-payments-from-peer` → aggregated list for the peer.
- `/api/channel/{channelId}/self-payments-to-channel` → list of self-payments into the channel.
- `/api/node/{pubkey}/self-payments-to-peer` → aggregated list for the peer.

### RatingController
- `/api/node/{pubkey}/rating` → `RatingDto` for peer (error 404 if none).
- `/api/channel/{channelId}/rating` → rating for the given channel.

### WarningsController
- `/api/node/{pubkey}/warnings` → list of warnings (offline %, no flow, rating...).
- `/api/channel/{channelId}/warnings` → warnings for the given channel.
- `/api/warnings` → all node and channel warnings.

### PaymentsController (Pickhardt Payments)
- `/api/payments/pay-payment-request/{paymentRequest}` (`GET` uses defaults, `POST` accepts `PaymentOptionsDto`).
- `/api/payments/to/{pubkey}/amount/{satoshis}` (`GET` default options; `POST` accepts options in body).
- `/api/payments/from/{source}/to/{target}/amount/{satoshis}` (same `GET`/`POST` pattern).
- `/api/payments/top-up/{pubkey}/amount/{satoshis}` and `/top-up/{pubkey}/amount/{satoshis}/via/{firstHop}` (both GET/POST variants, POST allows options, used to increase local balance).
- `/api/payments/reset-graph-cache` → invalidates the cached graph used for pathfinding.

### LegacyController
- `/legacy/open-channels/pretty` → human readable list of open channels.

## HTML UI Routes
Handled by controllers under `ui` module using Thymeleaf templates.
- `/` → dashboard showing open channels and peers (`DashboardController`).
- `/channel` or `/channels` → table of all open channels.
- `/node` or `/nodes` → table of all peers.
- `/channel/{channelId}` → detailed view of a channel (`ChannelDetailsController`).
- `/node/{pubkey}` → detailed view of a node (`NodeDetailsController`).
- `/pending-channels` → list of pending open/close channels.
- `/search?q=…` → search open channels or nodes by ID/pubkey/alias.
- `/status` → page with sync status and other runtime info.

## Domain Models
Below is the full list of classes found under `model/src/main/java/de/cotto/lndmanagej/model` with a short description of what they represent. Many are simple Java `record` types, used in both persistence and DTOs.

- `BalanceInformation` – six `Coins` values representing local/remote balance, reserve and available funds (see file for details).
- `BasicRoute` – describes a route in the channel graph with total amount and fee info.
- `BreachForceClosedChannel` – representation of a breach-closed channel, built via `BreachForceClosedChannelBuilder`.
- `Channel` – base type with `ChannelId`, channel point, capacity and two pubkeys.
- `ChannelCoreInformation` – immutable tuple of `ChannelId`, `ChannelPoint` and capacity.
- `ChannelDetails` – aggregation of `LocalChannel`, alias, `BalanceInformation`, `OnChainCosts`, `PoliciesForLocalChannel`, `FeeReport`, `FlowReport`, `RebalanceReport`, warnings and optional rating.
- `ChannelId` – wrapper for short channel id with helper parsing from compact forms.
- `ChannelIdAndMaxAge` – channel ID paired with an age for queries (used when retrieving transactions).
- `ChannelIdParser` – utility to parse channel IDs from multiple formats.
- `ChannelIdResolver` – uses gRPC to translate `ChannelPoint` to `ChannelId`.
- `ChannelPoint` – txid + output index.
- `ChannelRating` – rating value and associated statistics.
- `ChannelStatus` – enum for OPEN, OPENING, WAITING_CLOSE, CLOSING, FORCE_CLOSING, CLOSED.
- `CloseInitiator` – enum for LOCAL, REMOTE, MUTUAL, BREACH.
- `ClosedChannel` – contains close height, initiator, force/breach flags etc., built via `ClosedChannelBuilder`.
- `ClosedOrClosingChannel` – superclass for closing/waiting close channels.
- `Coins` – immutable long value plus helpers (satoshis, millisatoshis, addition/subtraction etc.).
- `CoinsAndDuration` – pair of `Coins` and Java `Duration`.
- `CoopClosedChannel` – cooperative close specific representation with builder.
- `DecodedPaymentRequest` – result of decoding a payment request via lnd.
- `Edge` – channel graph edge with capacity and pubkeys.
- `EdgeWithLiquidityInformation` – adds lower/upper liquidity bounds and in-flight amount.
- `FailureCode` – enumeration of lnd failure codes.
- `FeeRateInformation` – fee rate and the time it was retrieved.
- `FeeReport` – coins earned vs sourced.
- `FlowReport` – coins forwarded/rebalanced/etc. (10 numeric fields; see source for names).
- `ForceClosedChannel` – representation of force-closed channel with sweep fees.
- `ForwardAttempt`/`PaymentAttemptHop` – detail about attempts to forward or pay.
- `ForwardFailure` – details about a forwarding failure with timestamp and code.
- `ForwardingEvent` – event from forwarding history (amounts, channels, timestamp).
- `HexString` – wrapper for hex-encoded byte arrays.
- `HtlcDetails` – properties of a settled HTLC (hash, amounts, preimage etc.).
- `LiquidityBounds` – lower/upper bound for channel liquidity.
- `LiquidityBoundsWithTimestamp` – same plus observation timestamp.
- `LiquidityChangeListener` – interface for components interested in liquidity updates.
- `LocalChannel` – extends `Channel` with open/close status.
- `LocalOpenChannel` – extends `LocalChannel` with `BalanceInformation` and policies.
- `MissionControlEntry` – wrapper for `lnd` mission control information.
- `Network` – enumeration MAINNET or TESTNET.
- `Node` – immutable alias + pubkey pair.
- `NodeDetails` – aggregate details about a peer including rating and online stats.
- `OnChainCosts` – open/close/sweep `Coins` values.
- `OnlineReport` – online percentage and number of changes over a window.
- `OnlineStatus` – enum ONLINE/OFFLINE.
- `OpenCloseStatus` – enum OPEN, CLOSED etc. for channels.
- `OpenInitiator` – enum for who opened a channel.
- `OpenInitiatorResolver` – resolves channel open initiator by checking local/remote pubkey.
- `Payment` – outgoing payment with amount, fees, timestamp and routes.
- `PaymentAttemptHop` – part of the gRPC response describing a hop in an attempt.
- `PaymentHop` – simple representation of a payment hop with channel and amounts.
- `PaymentListener` – interface for components that react to newly seen payments.
- `PaymentRoute` – sequence of `PaymentHop` objects with total amount and fees.
- `PeerRating` – rating value for a peer plus analysis period.
- `PendingOpenChannel` – local channel waiting for confirmation.
- `PoliciesForLocalChannel` – local + remote `Policy` objects.
- `Policy` – base fee, ppm, CLTV delta and disabled flag.
- `PrivateResolver` – determines whether a channel is private.
- `Pubkey` – 33 byte compressed pubkey wrapper.
- `PubkeyAndFeeRate` – tuple of pubkey and long fee rate.
- `Rating` – generic rating value.
- `RebalanceReport` – amounts and costs for rebalances.
- `Resolution` – information about HTLC/on-chain resolutions during close.
- `Route` – combination of hops with CLTVs and amounts.
- `RouteHint` – hint for path finding (channel, base fee, ppm, cltv delta).
- `SelfPayment` – correlated invoice + payment (for rebalances).
- `SelfPaymentRoute` – route for a self payment (outbound/inbound channel list etc.).
- `SettledForward` – forwarding event that is known to be settled.
- `SettledInvoice` – invoice that has been paid with details (memo, keysend message, etc.).
- `TransactionHash` – 32-byte hex encoded string.
- `WaitingCloseChannel` – local channel waiting to close.
- `warnings/…` – hierarchy of warning types. Examples:
  - `NodeOnlinePercentageWarning` – triggered if online percentage below threshold.
  - `NodeOnlineChangesWarning` – triggered if too many status changes.
  - `NodeNoFlowWarning` – triggered if no flow for N days.
  - `NodeRatingWarning` – rating below threshold.
  - `ChannelBalanceFluctuationWarning` – large swings in channel balance.
  - `NodeWarnings` and `ChannelWarnings` aggregate lists of the above.

## Database Schema
Flyway migration `V1_0_0__base.sql` defines the core tables.
- `balances(channel_id,timestamp,local_balance,local_reserved,remote_balance,remote_reserved)`.
- `forwarding_events(event_index,amount_incoming,amount_outgoing,channel_incoming,channel_outgoing,timestamp)` plus indexes on incoming/outgoing channel ids.
- `online_peers(pubkey,timestamp,online)` – boolean status snapshots.
- `payments(payment_index,fees,hash,timestamp,value)` with indexes on `hash`.
- `payment_routes(route_id)` and `payment_route_hops(route_id,channel_id,amount,hops_order)`; `payments_routes` links payments to their ordered routes.
- `private_channels(channel_id,is_private)`.
- `settled_invoices(add_index,amount_paid,hash,keysend_message,memo,received_via,settle_date,settle_index)` with indexes.
- `transactions(hash,block_height,fees,position_in_block)`.
Other migrations add more indexes and columns as the project evolved.

## Configuration
`application.properties` sets defaults:
- HTTP server on `127.0.0.1:8081`.
- PostgreSQL datasource at `jdbc:postgresql://localhost:5432/lndmanagej` (user `bitcoin`).
- Caching and scheduling thread pool size.
- Rate limiters and circuit breakers for external APIs `Bitaps` and `Blockcypher` using Resilience4j.
- Path to reloadable configuration file: `${user.home}/.config/lnd-manageJ.conf`.
`application-h2.properties` uses H2 database and renames Flyway scripts with prefix `HTWO`.
The example configuration file `example-lnd-manageJ.conf` documents many tunables including warning thresholds, pickhardt-payments settings, top-up parameters, rating calculation days, etc.

## Background Data Collection
Each statistics module schedules an updater:
- `BalancesUpdater` (module `balances`) – every 5 minutes; writes to `balances` table via DAO.
- `OnlinePeersUpdater` (module `onlinepeers`) – checks peer online/offline status periodically and records into `online_peers` table.
- `ForwardingHistory` (module `forwarding-history`) – periodically polls lnd for forwarding events and stores them.
- `Transactions` (module `transactions`) – downloads missing transaction details from external APIs and caches them.
- `Payments` and `SettledInvoices` (modules `payments` and `invoices`) – fetch full histories from lnd on startup and then poll for new entries every 10 seconds.
- `SelfPayments` (module `selfpayments`) – correlates payments with invoices to compute self-payments for rebalancing.

## Caching
Many services wrap gRPC and database calls with Caffeine caches (via `CacheBuilder`). Expiry/refresh durations are tuned per use case (e.g. node alias cached for minutes, channel balance for <1s). `IniFileReader` also caches configuration file contents for 5–10 seconds to support live reload.

## Rating Algorithm
`rating.md` details how channel and node ratings accumulate data from payments, fees, rebalance support, and potential earnings. The values are normalized by channel age and average local balance. A configurable threshold (default 1000) triggers `NodeRatingWarning` when a peer’s rating is too low.

## Pickhardt Payments
Documented extensively in `PickhardtPayments.md`. The feature enables pathfinding and payment splitting based on liquidity information. The configuration file controls whether it is enabled, quantization (default 10k sat), number of piecewise linear approximations, CLTV limit, and whether to incorporate lnd mission control data. The `PaymentsController` exposes endpoints to compute multi-path payments, pay invoices, perform “top-up” operations, and reset the cached graph.

## HTML UI
The UI module uses Spring MVC controllers returning Thymeleaf templates. Pages include a dashboard, channel list, node list, pending channels, search results, and channel/node detail pages. A `StatusInterceptor` adds general status info to each request (synced to chain, block height). `ui-demo.sh` starts a demo using an H2 DB and fake data.

## Re-Implementation Notes
To port this project to another language you will need to:
1. Recreate the database schema including all tables and indexes.
2. Implement gRPC clients for lnd methods used throughout (`getinfo`, `listchannels`, `closedchannels`, `listpayments`, `listinvoices`, etc.).
3. Provide scheduled collectors for balances, online status, forwarding events, transactions, payments and invoices.
4. Cache frequently requested data with similar expiry semantics.
5. Compute warnings and ratings based on collected data using the formulas in `rating.md` and configuration thresholds.
6. Offer a REST API following the exact endpoints and JSON structures described above.
7. Replicate the pickhardt-payments module for advanced path finding, honoring the configuration values.
8. Optionally rebuild the UI layer using your language’s web framework and the same URI structure.

