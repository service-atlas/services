# Service Atlas Backend
![Coverage](https://img.shields.io/badge/Coverage-82.1%25-brightgreen)
![CodeRabbit Pull Request Reviews](https://img.shields.io/coderabbit/prs/github/service-atlas/backend?utm_source=oss&utm_medium=github&utm_campaign=service-atlas%2Fbackend&labelColor=171717&color=FF570A&link=https%3A%2F%2Fcoderabbit.ai&label=CodeRabbit+Reviews)

A RESTful API service designed to map dependencies between services and provide basic information about services in your ecosystem.

Service Atlas began as an exploration of Go and has evolved into an open-source platform for modeling service dependencies, ownership, releases, and operational risk. It is intended for internal or trusted environments and should be deployed with appropriate authentication and network controls.

## Overview

This API allows you to:

- Track services and their metadata (name, description, GitHub repo, etc.)
- Map dependencies between services
- Associate releases with services
- Query service relationships and dependencies

## Features

- Create, read, update, and delete services
- Map dependencies between services
- Track service metadata such as:
    - Name
    - Description
    - Database dependencies
    - GitHub repository
- Associate releases with services
- Ability to add technical debt to a service
- Classify services by operational criticality using Service Tiers (1–4)

## Design Philosophy

### What Service Atlas Is

Service Atlas is a tool for understanding and surfacing relationships between systems. It exists to answer one core question: *if something changes or fails here, what else is affected?*

Debt and release tracking exist solely to support risk analysis — they are not project management tools or replacements for your work tracking software.

### What Service Atlas Is Not

**It is not an infrastructure catalog.** An API gateway and the lambdas behind it are one service — the thing your team ships and operates as a cohesive unit. The underlying cloud resources that implement it are irrelevant to the graph.

**It is not a CMDB.** Service Atlas models what engineers think about and own, not every resource that exists in your cloud account. A catalog of 10,000 items where no one can find the 50 things they care about has negative value.

**It is not an incident management system.** It informs incident response by surfacing blast radius and ownership, but it does not replace your alerting, on-call, or postmortem tooling.

### Design Principles

- **Simple by design** — a service requires only a name and a type to exist. Complexity should never be a barrier to adoption.
- **Logical over physical** — model systems as engineers understand them, not as infrastructure defines them.
- **Relationships over inventory** — the graph is the value. A service in isolation is just a record.
- **Advisory not prescriptive** — risk scores support human judgment. They do not enforce policy or predict outcomes.
- **Engineer-first** — value should flow to the people doing the work, not only upward to leadership.


## What is a "Service"
A service is a distinct, independently deployable component of a larger platform. The unit of meaning is logical, not physical — it is the thing your team reasons about, names, owns, and ships.
Service was chosen as the initial use case for this api was to catalog microservices and relations between them. Services can `depend_on` other services and have `releases` associated with them

```mermaid
flowchart LR
    id1((Service A)) -- Depends On --> id2((Service B))
    id1 -- Released --> r1((Release 1))
    t1((Team A)) -- Owns --> id1

```

## Debt
This application supports the recording and tracking of categorized technical debt as part of the database. Technical debt can fall into the categories below.

### Debt Types
| Type           | Notes                                                         |
|----------------|---------------------------------------------------------------|
| code           | code smells or localized poor code quality                    |
| documentation  | lack of documentation about app purpose, how tos, etc         |
| testing        | lack of types of testing                                      |
| architecture   | issues with design choices that affect the entire application |
| infrastructure | issues with the infrastructure stack the app runs on          |
| security       | security issues, such as using packages past EOL              |

### Debt Statuses
| Status      | Notes                                  |
|-------------|----------------------------------------|
| pending     | a new debt item                        |
| in_progress | actively being worked on               |
| remediated  | a debt item that is no longer an issue |

### Why track technical debt?
I think adding technical debt (or code rot) is a useful way to track and quantify issues with services. Things that are in your work tracking software (Jira, etc.)
may or may not always be picked up in a reasonable timeframe and may not be easily associated with the service in question.

## Dependency Interaction Types

### Summary
Enriching `DEPENDS_ON` with interaction type classification to surface logical data flows and engineering intent. Rather than introducing a separate relationship type to distinguish logical data flows from operational dependencies, interaction types enrich the existing dependency graph with a classification that describes *why* one service depends on another.

### Classification Types

| Type | Default | Description |
|---|---|---|
| `data` | Yes | A direct, synchronous exchange of meaningful domain data between two services. Represents logical data flow. |
| `security` | No | A dependency on a service that handles authentication, authorisation, or request routing (e.g. Auth Service, Reverse Proxy). |
| `performance` | No | A dependency on a service that exists to optimise speed or reduce load (e.g. Redis cache, Elasticsearch). |
| `async` | No | A dependency where the interaction is fire-and-forget or event-driven (e.g. RabbitMQ, Kafka). |
| `config` | No | A dependency on a service that provides configuration, feature flags, or secrets at runtime (e.g. Vault, Feature flag service). |

### Scope
- New `interaction_type` field on the `DEPENDS_ON` relationship
- Validation of allowed interaction types
- Default behavior for existing relationships: if unset, treat as `data`
- API support for filtering dependencies by `interaction_type`

## Service Tiers (Criticality Classification)

### Summary
Service Tiers introduce a `tier` attribute on `Service` to represent how critical a component is to the overall operation of the platform. This helps reason about impact, prioritize work, and understand dependency risk across the system.

### Tier Model
Use 4 tiers, where Tier 1 is the most critical.

- Tier 1 — Mission Critical
  - Core to the platform’s primary purpose
  - Outage results in total or near-total platform failure
  - High customer, revenue, or availability impact
- Tier 2 — Business Critical
  - Platform remains partially functional if down
  - Significant degradation of key features or workflows
  - High user impact, but not a full outage
- Tier 3 — Supporting
  - Enhances or supports core functionality
  - Failures are noticeable but tolerable short-term
  - Often asynchronous or auxiliary services
- Tier 4 — Non-Critical / Auxiliary
  - Minimal operational impact if unavailable
  - Internal tools, dashboards, or experimental components

### Scope
- New `tier` field on the Service model
- Validation of allowed tier values (1–4)
- Default behavior for existing services: if unset, treat as Tier 3
- API support for creating, updating, and querying services by `tier`


## Neo4j Data Structure
Services are created under a `Service` object, while releases are created under a `Release` object.
Services can have a `DEPENDS_ON` relationship that may have a `version` and an `interaction_type` as part of the relationship.
Services `RELEASED` a `Release`
Services `OWNS` a `Debt`

Services also include a `tier` property to indicate operational criticality, validated to be an integer from 1 (most critical) to 4 (least critical). If not set, it defaults to Tier 3 (Supporting) behavior.

`DEPENDS_ON` relationships include an `interaction_type` property to classify the nature of the dependency (defaulting to `data`).

Releases will always have a date; releases without a date are assigned `now()` as the date. Releases may have an associated url, a version, or both, but require at least the url or a version to be present.

## Installation

### Prerequisites

- Go 1.26 or higher
- Neo4j database
- Docker and Docker Compose (optional, for local development)

### Using Docker Compose

1. Clone the repository
2. Start the Neo4j database:
   ```sh
   docker-compose up -d
   ```
3. Set the required environment variables:
   ```sh
   export SECRETS_PROVIDER=env
   export DB_URL=neo4j://localhost:7687
   export DB_USERNAME=neo4j
   export DB_PASSWORD=password
   ```
   *Note: These environment variables use the default `env` secrets provider. See [Configuration](#configuration) for alternative providers.*
4. Build and run the application:
   ```sh
   go build -o service-atlas ./cmd/service-atlas
   ./service-atlas
   ```

## Configuration

The application is configured using environment variables. The primary configuration relates to database connectivity and how secrets are managed.

### Core Settings

- `SECRETS_PROVIDER`: Defines how database credentials are retrieved. Options: `env` (default), `aws`.
- `LOG_LEVEL`: Controls logging verbosity. Options: `debug`, `info` (default), `warning`, `error`.
- `LOG_HEALTH`: Controls whether requests to the `/health` endpoint are logged. Defaults to `false`. Set to `true` to enable health check logs.

### Authentication (OIDC)

Authentication is optional. To enable OIDC-based authentication, all of the following environment variables must be set:

- `OIDC_ISSUER`: The URL of the OIDC issuer (e.g., `https://accounts.google.com`).
- `OIDC_AUDIENCE`: The expected audience for the tokens.
- `OIDC_MCP_CLIENT_ID`: The client ID used for MCP-side authentication.

If `OIDC_ISSUER` or `OIDC_AUDIENCE` are missing, authentication will be disabled.

#### MCP Authentication & Discovery

The service provides a discovery endpoint at `/auth/mcp/config` which helps MCP (Model Context Protocol) clients understand the required authentication configuration.

When OIDC is enabled, this endpoint returns:
- `auth_mode`: "enabled" (or "disabled"/"none" if configuration is incomplete)
- `oidc`: Contains the `issuer`, `client_id` (from `OIDC_MCP_CLIENT_ID`), and `audience` (from `OIDC_AUDIENCE`).

This allows the MCP client to automatically configure itself to obtain a JWT from the correct issuer with the correct audience and client ID, enabling secure communication with the Service Atlas API.

#### JWT Claims

When authentication is enabled, the middleware expects and extracts user information from the following namespaced claims in the JWT:

- `service-atlas:name`: The full name of the user.
- `service-atlas:email`: The email address of the user.

If neither is present, it falls back to the `sub` (subject) claim for identification.

### Database Connectivity & Secrets Management

Service Atlas supports multiple "Providers" for retrieving database credentials.

#### Environment Variable Provider (Default)

Used when `SECRETS_PROVIDER` is set to `env` or is unset.

- `DB_URL`: URL of the Neo4j database (e.g., `neo4j://localhost:7687`).
- `DB_USERNAME`: Username for Neo4j authentication.
- `DB_PASSWORD`: Password for Neo4j authentication.

#### AWS Secrets Manager Provider

Used when `SECRETS_PROVIDER` is set to `aws`. This provider retrieves a JSON secret from AWS Secrets Manager.

- `DATABASE_SECRET_NAME`: The name or ARN of the secret in AWS Secrets Manager.
- Standard AWS environment variables (e.g., `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) must be configured to allow access.

**Expected Secret Format:**
The AWS secret must be a JSON object with the following keys:
```json
{
  "url": "neo4j://your-neo4j-host:7687",
  "username": "neo4j",
  "password": "your-password"
}
```

The server listens on port 8080 by default.

### CORS Configuration

The API enables CORS via middleware. You can control allowed origins and methods via the `CORS_CONFIG` environment variable. If not set or invalid, the following default configuration is used:

```json
{
  "AllowedOrigins": ["*"],
  "AllowedMethods": ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
  "AllowedHeaders": ["Accept", "Authorization", "Content-Type", "X-CSRF-Token"],
  "AllowCredentials": false
}
```

To provide a custom configuration, set `CORS_CONFIG` to a JSON string with the same shape. Examples:

- Allow a single origin with credentials:

```bash
export CORS_CONFIG='{"AllowedOrigins":["https://example.com"],"AllowedMethods":["GET","OPTIONS"],"AllowCredentials":true}'
```

- Allow multiple specific origins and common methods:

```bash
export CORS_CONFIG='{"AllowedOrigins":["https://app.example.com","https://admin.example.com"],"AllowedMethods":["GET","POST","PUT","DELETE","OPTIONS"],"AllowCredentials":true}'
```

Notes:
- The value must be valid JSON; invalid JSON will be ignored and the default will be used.
- `AllowedOrigins` accepts `"*"` to allow all origins or a list of exact origin URLs.
- **Security Note**: If `AllowedOrigins` contains a wildcard (`*`), `AllowCredentials` will be automatically forced to `false` regardless of the configuration value to prevent unsafe cross-origin credential exposure.
- Only the specified HTTP methods are allowed for CORS preflight and actual requests.

## API Endpoints

For more information on endpoints, see the [Bruno Collection](./HTTP_COLLECTION).
Detailed discussions and RFCs can be found on GitHub, for example: [RFC: Dependency Interaction Types](https://github.com/service-atlas/backend/discussions/178).
