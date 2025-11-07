# TIP-XXXX: Tari Tapplets

```
TIP Number: XXXX
Title: Tari Tapplets
Status: Draft
Author(s): stringhandler
Created: 2025-11-06
```

## Abstract

Tapplets are small, sandboxed plugins that enable developers to build decentralized applications on the Tari blockchain with strong privacy guarantees. This proposal introduces a system where tapplets can store and retrieve encrypted data on-chain using one-sided transactions with data encoded in memo fields. Each tapplet operates within an isolated child account derived from a parent wallet account, using cryptographically derived view keys to ensure that only authorized parties can access tapplet-specific data. The system supports Lua-based scripting for ease of development and provides a registry-based distribution mechanism. Tapplets enable use cases such as password managers, encrypted note storage, and other privacy-sensitive applications while maintaining the security and decentralization benefits of blockchain storage.


## Motivation

The Tari base layer blockchain currently lacks a standardized framework for developers to build privacy-preserving decentralized applications that utilize on-chain storage. While the base layer provides secure and immutable storage, there is no straightforward mechanism for developers to store encrypted, application-specific data or extend wallet functionality in a modular way.

This proposal introduces **Tapplets** — a standardized plugin framework that enables developers to create privacy-preserving applications and wallet extensions that interact directly with the Tari blockchain.

#### Objectives

1. **Application-Specific Data Storage** – Provide a mechanism for applications to store encrypted data on-chain, ensuring sensitive information is not exposed to blockchain observers.
2. **Privacy-First Applications** – Enable use cases such as password managers, encrypted notes, and private data stores that benefit from Tari’s immutability and availability while maintaining user privacy.
3. **Plugin Extensibility (Tapplets)** – Define a standardized plugin system allowing third-party developers to extend wallet functionality using Lua scripting, without requiring complex UI development.
4. **Cross-Platform Execution** – Allow tapplets to run in both graphical and terminal wallet environments, supporting desktop, mobile, and even non-Tari applications for broader interoperability.
5. **Account Isolation** – Ensure each tapplet operates within an isolated storage context to prevent data leakage between applications.

#### Benefits

* **Developers:** Gain access to a simple, standardized API for building privacy-preserving dApps and wallet extensions.
* **Users:** Can securely store and manage private data with confidence that it remains encrypted and access-controlled.
* **Ecosystem:** Encourages third-party development, enabling a plugin marketplace and greater extensibility for Tari wallets.
* **Privacy Advocates:** Provides a practical example of privacy-preserving application design built on a public blockchain.

> Note that tapplets are different from smart contracts. Smart contract allow executing logic on Tari validator nodes. Tapplets allow 
execution of custom logic on a wallet client. In most cases these will be adapters that decode and present data stored in transaction fields.

## Specification

#### 1. Child Account System (Single Table Inheritance)

Tapplets use a child account model where each tapplet installation creates a derived account. The tapplet's data is stored using a viewkey 
derived from a domain specific hash of the account viewkey and the tapplet's public key.


#### 2. Key Derivation

Tapplet-specific view keys are derived using BLAKE2b-512:

```
tapplet_view_key = BLAKE2b-512(
    "tapplet_storage_address" ||
    parent_view_key ||
    tapplet_public_key
)
```


This ensures:
- Each tapplet has a unique view key
- Data is isolated from other tapplets
- Only those with both parent keys and tapplet public key can decrypt

> NOTE: The tapplet registry should maybe also be included, so that malicious developers cannot reuse the `tapplet_public_key` across multiple tapplet registrations. Including the registry identifier in the key derivation ensures that each tapplet's view key is unique, even if the same public key is used elsewhere.

##### The risks of sharing of view keys
In some situation, a user might upload their view key to a custodial wallet. Doing so would allow the custodial provider to see
any data that was stored by tapplets, although they would have to know which tapplets the user installed. 

This can be addressed by adding a password for installing a tapplet. In this case, the tapplet view key can also be derived by including a user specific password. 
```
tapplet_view_key = BLAKE2b-512(
    "tapplet_storage_address" ||
    parent_view_key ||
    tapplet_public_key || 
    password
)
```

Future installs of the tapplet would require knowledge of the password to retrieve any data stored. 

One potential use would be a password manager tapplet. Using a master password, the user can safely 
provide the view key to other users without revealing any of the passwords stored by the tapplet.

#### 3. Data Storage Format

Data is stored in transaction memo fields using the format:

```
t:"<slot_name>","<value>"
```

**Parsing Pattern:**
```regex
^t:"([^"]+)","((?s:.*?))"$
```

Where:
- `slot_name`: Application-defined storage key (e.g., "default", "passwords")
- `value`: Application-specific data (can contain any string, including newlines)

Any double quotes should be escaped using two double quotes. This basically matches the CSV format.

> Note: in future, binary memos may be stored as well.

#### 4. Core API Methods

Tapplets written in Lua can interact with the host wallet through a predetermined API. Apart from this API, the executing Lua is sandboxed using the [mlua](https://docs.rs/mlua/latest/mlua) crate. Any non-rust implementations should also ensure the Lua is sandboxed.

**Sandboxing:**
- Lua execution environment restricts system access
- API access limited to defined methods
- No direct filesystem or network access from tapplet code


##### Motivation for Lua

Originally, I intended to use WASM for tapplets. While WASM allows for sandboxing, it does not allow easy support for calling functions with strings and arrays (e.g. public keys as bytes), and would require more plumbing to call simple methods.

Lua is built for small plugins and is a simpler language, making it a good choice.


##### Interface

> Note: this interface is intended to be expanded in future and will include API methods for both Minotari and the Ootle.

**MinotariTappletApiV1 Interface:**

**`append_data(slot: &str, value: &str)`**
- Creates a one-sided transaction to the tapplet storage address
- Encodes data as `t:"<slot>","<value>"` in memo field
- Uses derived tapplet view key for encryption
- Returns unsigned transaction for signing

**`load_data_entries(slot: &str)`**
- Queries outputs for the child account
- Parses memo fields matching the slot pattern
- Returns array of all stored values in chronological order

#### 5. Tapplet Structure

**manifest.toml:**
```toml
name = "tapplet_name"
version = "0.1.0"
friendly_name = "Human Readable Name"
description = "Tapplet description"
publisher = "<publisher_key_hex>"
public_key = "<tapplet_public_key_hex>"
permissions =["append_data", "load_data"]

[api]
methods = ["method1", "method2"]

[api.method1]
description = "Method description"
[api.method1.params]
param_name = { type = "string", description = "..." }

```

**Implementation File (Lua):**
```lua
function method_name(args)
    -- Access parameters
    local value = args.param_name

    -- Store data
    minotari_append_data("slot", "data")

    -- Load data
    local entries = minotari_load_data_entries("slot")

    return { result = "value" }
end
```


##### Permissions
Each API method may require one or more permissions. The permissions should be stated in the manifest file, and users should be prompted during install whether to grant the permissions or not. 

When an API method is called, the permissions must be checked again to make sure the user has accepted them. Wallets should prompt the user and store their acceptance or declination. It is suggested to use a similar approach to mobile phone applications:

- Accept this time
- Accept always
- Decline

#### 6. Transaction Protocol

**Storage Transaction Flow:**
1. Tapplet calls `minotari_append_data(slot, value)`
2. System creates one-sided transaction:
   - Recipient: Derived tapplet storage address
   - Memo: `t:"<slot>","<value>"`
   - Encryption: Uses tapplet-specific view key
3. Transaction submitted to blockchain
4. Output becomes queryable via child account

**Retrieval Flow:**
1. Tapplet calls `minotari_load_data_entries(slot)`
2. System queries outputs for child account
3. Decrypts memo fields using tapplet view key
4. Filters by slot pattern
5. Returns array of values

#### 7. Registry System

A default registry should be hosted in the Tari Github organization, but other registries can be added by the user at their own risk.

Installing a tapplet should be a simple command in the wallet, and can be done by pulling the latest registry and copying the folder for the 
tapplet into a local file. 

> Note: it is not clear whether this is possible on mobile devices at this time.

**Default Registry:**
- GitHub-based: `https://github.com/tari-project/tapplet-registry`
- Structure: Git repository with tapplet directories

**Registry Entry:**
```
registry/
|--tapplets/
|  |--tapplet_public_key/
|  |  |--version/
|  |  |  |--package/
|  |  |  |  |--manifest.toml
|  |  |  |  |--implementation.lua
|  |  |  |--sigs.json
|  |  |--README.md
|--publishers/
|  |--publisher_public_key/
|  |  |--publisher_manifest.toml
```

#### 8. Commands
Below is a list of example commands that a CLI wallet should implement. Wallets with a UI should also implement these in some form.

```bash
# Registry management
tapplet fetch                              # Update registries
tapplet search --query <query>             # Search for tapplets

# Installation
tapplet install --name <name>              # Install from registry
tapplet install --path <path>           # Install from local path
  
# Management
tapplet list                               # List installed tapplets

# Execution
tapplet run \
  --name <tapplet_name> \
  --method <method_name> \
  --args key1=value1 key2=value2 \
  --password <password>
```



## Rationale


#### Lua vs WASM

**Decision**: Use Lua as the primary scripting language (with WASM support retained)

**Rationale**:
- **Simpler Development**: Lua has a gentler learning curve than WASM toolchains
- **Dynamic Typing**: Better suited for quick prototyping and plugin development
- **Easy Function Calling**: Direct function invocation without complex FFI
- **Lightweight**: Small runtime footprint suitable for embedded execution
- **Proven**: Widely used in plugin systems (Redis modules, game mods)

**Alternative Considered**: WASM-only approach
- More complex build process
- Steeper learning curve for developers
- Better performance for compute-intensive tasks
- Retained as option for performance-critical tapplets


#### Memo Field Storage

**Decision**: Store data in transaction memo fields with custom encoding

**Rationale**:
- **Existing Infrastructure**: Memo fields already encrypted in Tari transactions
- **No Protocol Changes**: Works with existing consensus rules
- **Privacy-First**: Automatic encryption via existing mechanisms
- **Flexible Format**: Can store arbitrary string data

**Format** (`t:"slot","value"`):
- `t:` prefix identifies tapplet data
- Quoted strings allow special characters
- Regex parsing with capturing groups

**Alternative Considered**: Custom transaction output types
- Would require consensus changes
- More complex implementation
- Better performance for large data
- May be considered for future versions

**Alternative Considered**: Storing as a custom smart contract on the Ootle
- May be considered for future versions
- Using tari and minotari transactions means this can be hidden among other transactions

#### Registry System

**Decision**: Git-based registry with TOML manifests

**Rationale**:
- **Decentralization**: Anyone can host a registry
- **Version Control**: Git provides built-in versioning
- **Auditability**: Full history of tapplet changes
- **Familiar**: Developers already know Git
- **No Central Authority**: Multiple registries can coexist

**Alternative Considered**: On-chain registry
- More decentralized but higher cost
- Immutable but cannot remove malicious tapplets
- May be considered for future versions


## Backwards Compatibility

Adding tapplets to the existing `minotari_console_wallet` is not recommended. I would suggest only adding it to the newer `minotari_cli` wallet. Below is a list of changes:

### Database Schema Changes

**Incompatibility**: Requires database migration to add child account support

**Severity**: Medium - Existing wallets need migration

**Migration Path**:
1. Migration `20251016122205_add_child_accounts.sql` adds:
   - `account_type` column (default: 'parent')
   - `parent_account_id` foreign key
   - `for_tapplet_name` column
   - `version` column
   - `tapplet_pub_key` column

2. Existing accounts automatically become 'parent' type
3. No data loss - all existing accounts remain functional
4. New tapplet installations create 'child' accounts

**Impact**: Minimal - Migration is automatic and non-breaking for existing functionality

### API Changes

**Incompatibility**: None

**Rationale**: Tapplets are a new feature, not a modification of existing APIs. All existing wallet functionality remains unchanged.

### Consensus Changes

**Incompatibility**: None

**Rationale**:
- Uses existing transaction format
- Memo fields already supported
- No changes to validation rules
- No new transaction types
- Fully compatible with existing nodes

### Forward Compatibility

**Tapplet Versioning**:
- Manifest includes version field
- Multiple versions can coexist
- Future API versions (MinotariTappletApiV2, etc.) can be added
- Old tapplets continue to work with original API

**Registry Evolution**:
- New registries can be added without breaking existing ones
- Manifest format extensible via TOML
- Backward-compatible changes preferred

## Test Cases

While tapplets do not affect consensus rules, the following test cases validate the implementation:

### 1. Key Derivation Tests

**Test**: Verify deterministic key derivation
```rust
#[test]
fn test_tapplet_key_derivation() {
    let parent_view_key = PrivateKey::random();
    let tapplet_pub_key = "0x123abc...";

    let derived1 = derive_tapplet_view_key(&parent_view_key, tapplet_pub_key);
    let derived2 = derive_tapplet_view_key(&parent_view_key, tapplet_pub_key);

    assert_eq!(derived1, derived2); // Deterministic

    let different_tapplet = "0x456def...";
    let derived3 = derive_tapplet_view_key(&parent_view_key, different_tapplet);

    assert_ne!(derived1, derived3); // Different per tapplet
}
```

### 2. Data Storage and Retrieval Tests

**Test**: Verify append and load operations
```lua
-- Lua test in tapplet
function test_storage()
    minotari_append_data("test_slot", "value1")
    minotari_append_data("test_slot", "value2")

    local entries = minotari_load_data_entries("test_slot")

    assert(#entries == 2)
    assert(entries[1] == "value1")
    assert(entries[2] == "value2")
end
```

### 3. Memo Field Parsing Tests

**Test**: Verify correct parsing of tapplet data format
```rust
#[test]
fn test_memo_parsing() {
    let memo1 = r#"t:"slot1","simple value""#;
    let memo2 = r#"t:"slot2","value with \"quotes\"""#;
    let memo3 = r#"t:"slot3","multiline\nvalue""#;

    assert_eq!(parse_tapplet_memo("slot1", memo1), Some("simple value"));
    assert_eq!(parse_tapplet_memo("slot2", memo2), Some(r#"value with "quotes""#));
    assert_eq!(parse_tapplet_memo("slot3", memo3), Some("multiline\nvalue"));
    assert_eq!(parse_tapplet_memo("wrong_slot", memo1), None);
}
```

### 4. Account Isolation Tests

**Test**: Verify child accounts are properly isolated
```rust
#[test]
fn test_account_isolation() {
    let parent_account = create_account("parent");
    let tapplet1_account = create_child_account(&parent_account, "tapplet1");
    let tapplet2_account = create_child_account(&parent_account, "tapplet2");

    // Store data in tapplet1
    append_data(&tapplet1_account, "slot", "data1");

    // Verify tapplet2 cannot see tapplet1's data
    let tapplet2_data = load_data_entries(&tapplet2_account, "slot");
    assert!(tapplet2_data.is_empty());
}
```

### 5. Installation Tests

**Test**: Verify tapplet installation from registry
```bash
# Install from registry
tapplet install --name password_manager --account-name default --password test123

# Verify installation
tapplet list | grep "password_manager"

# Run installed tapplet
tapplet run --account-name default --name password_manager \
  --method greet --args name="World" --password test123
```

### 6. Security Tests

**Test**: Verify unauthorized access is prevented
```rust
#[test]
fn test_unauthorized_access() {
    let account1 = create_account("account1");
    let tapplet_account = create_child_account(&account1, "tapplet1");

    append_data(&tapplet_account, "private", "secret");

    // Different parent account cannot decrypt
    let account2 = create_account("account2");
    let result = try_load_data(&account2, &tapplet_account.address);

    assert!(result.is_err()); // Cannot decrypt
}
```

### 7. Example Tapplet: Password Manager

**Location**: `data/data/tapplet_cache/installed/password_manager/`

**Test**: Full end-to-end workflow
```bash
# Save password entry
tapplet run --account-name default --name password_manager \
  --method save --args site="github.com" username="user" password="pass123" \
  --password test123

# Load password entries
tapplet run --account-name default --name password_manager \
  --method load --password test123

# Expected output: List of password entries including github.com
```

## Implementation

An example implementation is provided as [PR8 on the minotari-cli repo](https://github.com/tari-project/minotari-cli/pull/8)

## References

Links to relevant resources:
- Related TIPs
- External documentation
- Research papers
- Code repositories

## Copyright

This document is released under the BSD 3-Clause License.
