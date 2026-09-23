# QUBCloud-FraudGuard

A stateless, horizontally-scalable fraud-screening platform for payment transactions, built as a multi-service cloud architecture project. Started from a small working base (a frontend, a single-check screening service, and a country-tally service) and expanded into a resilient, monitored, multi-cloud system.

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │        QPC (Kubernetes cluster)          │
                        │                                           │
   ┌──────────┐         │   ┌─────────────┐      ┌─────────────┐  │
   │ Frontend │─────────┼──▶│   fgproxy   │◀─────│  fgmonitor  │  │
   └──────────┘         │   │(reverse     │      │(health      │  │
        │                │   │ proxy +     │      │ checks +    │  │
        │ /api/save      │   │ circuit     │      │ alerting)   │  │
        │ /api/recall    │   │ breaker)    │      └─────────────┘  │
        │                │   └──────┬──────┘                       │
        │                │          │                               │
        │                │  ┌───────┼────────┐                     │
        │                │  ▼       ▼        ▼                     │
        │                │ Screening Reputation Country-Tally       │
        │                │ Service   Service    Service             │
        │                │ (Flask)   (Flask)    (Node.js)           │
        │                └───────────────────────────────────────┘  │
        │                                                            │
        ▼                                                            
┌──────────────┐                                                    
│   fgstore     │  ← GCP Cloud Run + Cloud Storage                  
│ (save/recall) │    (the only stateful component)                  
└──────────────┘                                                    
```

All services on QPC are fully stateless — each request is processed purely from its inputs, with nothing held between calls. The single stateful feature (saving and recalling a batch of transactions) is deliberately isolated to its own service on a public cloud provider, since the private Kubernetes cluster has no persistent storage.

## Services

| Service | Stack | Role |
|---|---|---|
| `fgfrontend` | Node.js (no framework) | Serves the web UI, relays authenticated save/recall calls server-side |
| `fgscreeningservice` | Python / Flask | Runs the fraud checks: amount threshold, blocked-card list, high-risk country, and a delegated card-reputation lookup |
| `fgreputationservice` | Python / Flask | Isolated "hard" check — simulates an external reputation lookup, split out so it can scale/fail independently of the fast local checks |
| `fgcountrytally` | Node.js | Counts transactions per country from a batch |
| `fgproxy` | Node.js | Custom reverse proxy in front of the screening/tally services — config-driven routing with hot-reload, and a per-route circuit breaker |
| `fgmonitor` | Node.js | On-demand and periodic health checking, with alert logging on failure |
| `fgstore` | Node.js (Express) | Save/recall API backed by Google Cloud Storage, deployed on Cloud Run |

## Key design decisions

- **Service decomposition**: the fast, local checks (threshold, blocklist, country) stay in the screening service; the slower, external-style reputation lookup is isolated in its own service so it can scale or fail independently without dragging down the fast path.
- **Configurable, resilient routing**: `fgproxy` reads its backend routes from a config file that can be hot-reloaded without a rebuild or restart, and falls back to the last-known-good config if a reload fails.
- **Circuit breaker**: after 3 consecutive failures to a backend, `fgproxy` opens the circuit and fails fast (immediate response) for a 15-second cooldown, rather than continuing to hang on a dead service.
- **Frontend failover**: the browser client tries each proxy endpoint in `PROXY_URLS` in order, so a single proxy instance going down doesn't take down the whole system.
- **Cross-cloud persistence**: the one stateful feature runs on GCP (Cloud Run + Cloud Storage) rather than on the private cluster, accessed via a server-side authenticated relay (browsers can't attach auth headers to CORS preflight requests against an IAM-protected Cloud Run service directly).

## Running locally

Each service is a standalone Docker container. From within a service's folder:

```bash
docker build -t <service-name> .
docker run -p <port>:<port> <service-name>
```

The frontend expects `PROXY_URLS` (comma-separated proxy addresses) and `STORE_URL` (the fgstore endpoint) as environment variables; `fgmonitor` and `fgproxy` expect a `PROXY_URL`/backend config pointing at a running proxy/screening stack.

## Context

Built for a postgraduate cloud computing module, starting from a small provided base system and expanded task-by-task into the architecture above, with each significant design choice made deliberately and evidenced against a running deployment.
