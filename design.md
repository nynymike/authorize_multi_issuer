# Design Document: authorize_multi_issuer Interface

## Overview

The authorize_multi_issuer interface is a Cedar-based authorization system that processes multiple JWT tokens from various issuers and maps them to Cedar entities for policy evaluation. The system is designed to be flexible and extensible, handling arbitrary token mappings (like "Acme::DolphinToken") without pre-defining all possible token types.

The interface accepts an array of token objects, validates them, extracts claims, and creates individual Cedar entities for each token. The system then joins tokens of the same type from the same issuer, aggregating compatible claims (like scopes) while preserving common claims, to create a single queryable token entity per issuer/type combination.

### Example input JSON

Developer sends an Array of all the tokens: 

```
[
  {
    "mapping": "Jans::Tx_token",
    "payload": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
  },
  {
    "mapping": "Jans::Tx_token",
    "payload": "ey97df42637ce64a1c9bb38067d3957054..."
  },
  {
    "mapping": "Jans::Access_Token", 
    "payload": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  },
  {
    "mapping": "Jans::Access_Token", 
    "payload": "ey84351930d3504880b4cec36c828ff7d7..."
  },
  {
    "mapping": "Jans::Passkey_Authenticator_Attestation", 
    "payload": "ey1b6cfMef21084633a77fae9dT1b6cefe..."
  },
  {
    "mapping": "Jans::Id_token", 
    "payload": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9..."
  },
  {
    "mapping": "Jans::Userinfo_token"
    "payload": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9..."
  }
]
```

## Architecture

### High-Level Flow

```
Input: Array of Token Objects
    ↓
Token Validation & Parsing
    ↓
Dynamic Cedar Entity Creation
    ↓
Token Collection Assembly
    ↓
Policy Evaluation
    ↓
Authorization Decision
```

### Core Components

1. **Token Processor**: Validates and parses JWT tokens
2. **Dynamic Entity Factory**: Creates Cedar entities for any token mapping type
3. **Token Collection Builder**: Assembles tokens into queryable collections
4. **Policy Engine**: Evaluates Cedar policies against individual tokens
5. **Response Handler**: Formats authorization decisions and error messages

## Components and Interfaces

### Input Interface

```typescript
interface TokenInput {
  mapping: string;  // e.g., "Jans::Access_Token", "Acme::DolphinToken"
  payload: string;  // JWT token string
}

interface authorize_multi_issuer {
  tokens: TokenInput[];
  resource?: ResourceContext;
  action?: string;
  context?: string;
}
```

### Token Processor

Use existing Cedarling token validation capabilities to 
- Validate JWT signature
- Check contents: e.g. `exp`, `nbf` 
- Extract raw claim (no interpretation)
- Check token status

### Dynamic Entity Factory

**Responsibilities:**
- Create Cedar entities for any token mapping type
- Convert JWT claims to Cedar-compatible attributes
- Handle unknown/custom token types dynamically

**Key Features:**
- No pre-defined entity types - creates entities based on mapping string
- Preserves all JWT claims as entity attributes
- Handles arbitrary claim structures
- Creates joined token entities with predictable field names

### Token Collection Builder

**Responsibilities:**
- Organize tokens into collections by issuer and token type
- Join tokens of the same type from the same issuer
- Create queryable token collections with flattened naming

**Key Features:**
- Groups tokens by issuer (e.g., all tokens with `iss` "idp.dolphin.sea" together)
- Joins tokens of the same type from the same issuer into single entities
- Aggregates compatible claims (like scopes) while preserving common claims
- Creates flattened field names for ergonomic policy syntax
- Supports N tokens of any type

## Data Models

### Dynamic Cedar Entity Approach

Instead of pre-defining entity types, the system creates Cedar entities dynamically based on the token mapping. This allows for arbitrary token types like "Acme::DolphinToken" without schema modifications.

### Generic Token Entity Structure

Each token becomes a Cedar entity with this flexible structure:

```cedar
entity Token = {
  "token_type": String,                  // e.g., "Jans::Access_Token", "Acme::DolphinToken"
  "jti": String,                         // Unique identifier for this token instance, hopefully the `jti`
  "issuer": String,                      // JWT issuer claim
  "claims": Record<String, EntityValue>, // All JWT claims as key-value pairs
  "validated_at": Long,                  // Timestamp when token was validated
};
```

### Token Collection Context

The Cedar context contains flattened collections with predictable naming for ergonomic policy syntax:

```cedar
entity TokenCollection = {
  // Format: {issuer_domain}_{token_type_simplified}
  "dolphin_sea_access_token": Token,           // Single joined access token from idp.dolphin.sea
  "dolphin_sea_id_token": Token,               // Single joined ID token from idp.dolphin.sea  
  "google_access_token": Token,                // Single joined access token from accounts.google.com
  "acme_dolphin_token": Token,                 // Single joined Acme::DolphinToken from idp.dolphin.sea
  "total_token_count": Long,
};
```

## Policy Examples

Here are Cedar policy examples using the flattened structure approach for ergonomic policy syntax:

### Basic Token Queries

```cedar
// Query tokens from specific issuer with claim check
permit(
  principal,
  action == Acme::Action::"GetFood",
  resource in Acme::Resources::"ApprovedDolphinFoods"
) when {
  context.token_collection has "dolphin_sea_access_token" &&
  context.token_collection.dolphin_sea_access_token.claims has "location" &&
  context.token_collection.dolphin_sea_access_token.claims.location == "miami"
};

// Query specific token type with scope
permit(
  principal,
  action == Jans::Action::"ReadProfile",
  resource
) when {
  context.token_collection has "google_access_token" &&
  context.token_collection.google_access_token.claims has "scope" &&
  "read:profile" in context.token_collection.google_access_token.claims.scope
};
```

### Multi-Token Scope Requirements

```cedar
// Require scopes from multiple tokens (can be from different issuers)
permit(
  principal,
  action == Jans::Action::"WriteDocument",
  resource
) when {
  // Check if any access token has write:documents scope
  (context.token_collection has "google_access_token" &&
   context.token_collection.google_access_token.claims has "scope" &&
   "write:documents" in context.token_collection.google_access_token.claims.scope) ||
  (context.token_collection has "corp_access_token" &&
   context.token_collection.corp_access_token.claims has "scope" &&
   "write:documents" in context.token_collection.corp_access_token.claims.scope)
};
```

### Joined Token Queries (Same Type, Same Issuer)

```cedar
// Use joined tokens for combined scope checking
permit(
  principal,
  action == Jans::Action::"ManageCalendar",
  resource
) when {
  context.token_collection has "google_access_token" &&
  context.token_collection.google_access_token.claims has "scope" &&
  "read:calendar" in context.token_collection.google_access_token.claims.scope &&
  "write:calendar" in context.token_collection.google_access_token.claims.scope
};
```

### Custom Token Type Handling

```cedar
// Handle arbitrary token mappings like Acme::DolphinToken
permit(
  principal,
  action == Acme::Action::"SwimWithDolphins",
  resource
) when {
  context.token_collection has "dolphin_sea_acme_dolphin_token" &&
  context.token_collection.dolphin_sea_acme_dolphin_token.claims has "clearance" &&
  context.token_collection.dolphin_sea_acme_dolphin_token.claims.clearance == "advanced"
};
```

### Multi-Issuer Validation

```cedar
// Require tokens from multiple specific issuers
permit(
  principal,
  action == Enterprise::Action::"AccessSecureResource",
  resource
) when {
  context.token_collection has "corp_id_token" &&
  context.token_collection has "security_access_token" &&
  context.token_collection.security_access_token.claims has "security_level" &&
  context.token_collection.security_access_token.claims.security_level == "high"
};
```

### Time-Based and Validation Policies

```cedar
// Ensure token is not expired
forbid(
  principal,
  action,
  resource
) when {
  context.token_collection has "dolphin_sea_access_token" &&
  context.token_collection.dolphin_sea_access_token.claims has "exp" &&
  context.token_collection.dolphin_sea_access_token.claims.exp <= context.current_time
};

// Require recent authentication
permit(
  principal,
  action == Sensitive::Action::"ViewPII",
  resource
) when {
  context.token_collection has "corp_id_token" &&
  context.token_collection.corp_id_token.claims has "auth_time" &&
  (context.current_time - context.token_collection.corp_id_token.claims.auth_time) < 300
};
```

### Field Naming Convention

The Token Collection Builder creates predictable field names using this pattern:
- `{issuer_domain}_{token_type_simplified}`

Examples:
- `idp.dolphin.sea` + `Jans::Access_Token` → `dolphin_sea_access_token`
- `accounts.google.com` + `Jans::Id_Token` → `google_id_token`
- `idp.dolphin.sea` + `Acme::DolphinToken` → `dolphin_sea_acme_dolphin_token`

### Token Joining Logic

When multiple tokens of the same type come from the same issuer, the Token Collection Builder:
1. Merges compatible claims (like scopes)
2. Preserves common claims (like location, subject)
3. Uses latest expiration time
4. Uses earliest issued time
5. Creates a single joined token entity

This enables policies to check for combined scopes across multiple tokens while maintaining clean, ergonomic syntax. 