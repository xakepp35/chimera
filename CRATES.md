<!-- artifact:"CRATES.md" v1.0.0 2026-09-23T18:36:00Z Architecture Топология Rust crates и границы ответственности Chimera -->

# Chimera — Crate Architecture

**Purpose:** определить закрытый состав Rust workspace, границы ответственности crates и разрешённый dependency DAG  
**Version:** 1.0.0  
**Owner:** xakepp35  
**Scope:** production-ready MVP Chimera  
**License:** Apache-2.0

---

# 1. Архитектурная модель

Chimera строится как набор малых ортогональных компонентов.

Базовое разделение:

```text
IDENTITY ≠ LOCATION ≠ TRANSPORT ≠ LINK ≠ ROUTE ≠ CIRCUIT ≠ DATA
```

Каждый уровень владеет только своей семантикой.

```text
                         chimera
                            │
             ┌──────────────┼───────────────┐
             │              │               │
          config         platform        adapters
             │              │               │
             └──────────────┼───────────────┘
                            ▼
                       chimera-node
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
       chimera-control             chimera-dataplane
              │                           │
       ┌──────┼────────┐                  │
       ▼      ▼        ▼                  ▼
     route  circuit   peer              circuit
       │      │                          │
       │      ▼                          ▼
       │     mux ◄────────────────────── mux
       │      │
       │      ▼
       │     link
       │      │
       │      ▼
       └──► carrier
```

Control plane принимает решения.

Data plane исполняет уже принятые решения.

**Control plane не участвует в обработке каждого пакета.**

---

# 2. Глобальные инварианты

## 2.1 Dependency direction

Dependency разрешена только вниз по DAG.

Ни один нижний crate не знает о верхнем.

Циклических crate dependencies нет.

---

## 2.2 Один владелец факта

Для каждой сущности существует один authoritative owner.

Примеры:

```text
UserId                  → chimera-types
cryptographic operation → chimera-crypto
identity semantics      → chimera-identity
carrier connection      → chimera-carrier
authenticated hop       → chimera-link
stream multiplexing     → chimera-mux
route decision          → chimera-route
multi-hop circuit       → chimera-circuit
packet forwarding       → chimera-dataplane
runtime reconciliation  → chimera-control
```

Один и тот же факт не моделируется повторно в нескольких crates.

---

## 2.3 Wire format не использует Serde

`serde`, YAML, JSON, CBOR и аналогичные generic serializers не участвуют в network wire protocol.

Wire parsing:

```text
explicit
bounded
canonical
versioned
reject-noncanonical
```

---

## 2.4 Никаких скрытых global state

Запрещены архитектурные зависимости от:

```text
global singleton
process-global mutable registry
implicit environment configuration
hidden thread-local protocol state
```

Все runtime dependencies передаются явно.

---

## 2.5 Bounded resources

Любая очередь имеет capacity.

Любой wire length имеет protocol maximum.

Любой peer/circuit/stream allocation имеет quota.

Любой receive path проверяет размер **до allocation**.

---

## 2.6 Data plane

Hot path не выполняет:

```text
YAML parsing
IAM lookup
route calculation
policy compilation
DNS resolution
filesystem I/O
key loading
configuration reconciliation
```

Data plane получает уже скомпилированный immutable/RCU-like state.

---

# 3. Полный workspace

Production MVP содержит следующие runtime crates:

```text
crates/
├── chimera-types
├── chimera-codec
├── chimera-device
├── chimera-crypto
├── chimera-crypto-default
├── chimera-identity
├── chimera-policy
├── chimera-carrier
├── chimera-peer
├── chimera-keystore
├── chimera-carrier-quic
├── chimera-carrier-h3
├── chimera-carrier-https
├── chimera-link
├── chimera-mux
├── chimera-route
├── chimera-circuit
├── chimera-dataplane
├── chimera-platform
├── chimera-control
├── chimera-node
└── chimera-config

cmd/
└── chimera
```

Это закрытый набор MVP.

**Если ответственность не перечислена в этом документе — она не входит в MVP.**

---

# 4. Dependency DAG

Топологическая сортировка:

```text
00 chimera-types
│
├── 01 chimera-codec
├── 02 chimera-device
├── 03 chimera-crypto
│   └── 04 chimera-crypto-default
│
├── 05 chimera-identity
│   ├── 06 chimera-policy
│   └── 09 chimera-keystore
│
├── 07 chimera-carrier
│   ├── 08 chimera-peer
│   ├── 10 chimera-carrier-quic
│   ├── 11 chimera-carrier-h3
│   └── 12 chimera-carrier-https
│
├───────────────┐
│               │
▼               ▼
13 chimera-link  08 chimera-peer
│                   │
▼                   │
14 chimera-mux       │
│                   ▼
│              15 chimera-route
│                   │
├───────────────────┤
▼                   ▼
16 chimera-circuit
│
├─────────────────────────┐
▼                         │
17 chimera-dataplane      │
│                         │
▼                         │
18 chimera-platform       │
                          │
              ┌───────────┘
              ▼
       19 chimera-control
              │
              ▼
        20 chimera-node

21 chimera-config

        adapters + node + config
                 │
                 ▼
             22 chimera
```

Прямая dependency означает только compile-time dependency.

Runtime communication между control/data plane идёт через bounded handles и immutable snapshots.

---

# 5. `chimera-types`

## Назначение

Самый внутренний crate.

Содержит только стабильные value types, используемые более чем одним слоем.

## Владеет

Полный scope:

```text
UserId
NodeId
ServiceId

PeerId
LinkId
CircuitId
StreamId
FlowId

CarrierId
EndpointId
RouteId

ProtocolVersion
ProtocolEpoch

CryptoSuiteId
SignatureSuiteId
KemSuiteId
AeadSuiteId

Timestamp
DurationMs

Role
Capability
CapabilitySet

Direction
Priority
TransportClass
```

Также:

- bounded numeric newtypes;
- parsing/formatting идентификаторов;
- canonical textual representation;
- ordering/hash/equality;
- protocol constants общего уровня.

## Не владеет

Нет:

```text
crypto operations
network I/O
wire codec
async runtime
identity trust
routing
configuration
serde YAML
filesystem
```

## Dependencies

```text
internal: none
```

Разрешены только минимальные utility dependencies уровня:

```text
bytes-compatible value types
smallvec if justified
thiserror
```

Но предпочтительно стандартная библиотека.

---

# 6. `chimera-codec`

## Назначение

Единый фундамент бинарного wire encoding.

Он знает **как кодировать**, но не знает **что означает сообщение**.

## Владеет

```text
Encoder
Decoder

FrameHeader
FrameType

VarInt
Length
Flags

DecodeLimits
EncodeLimits

EncodeError
DecodeError
```

Операции:

```text
encode_u8/u16/u32/u64
encode_varint
encode_bytes
encode_bounded_bytes

decode_*
```

Гарантии:

```text
canonical encoding
bounded decoding
zero-copy slices where possible
no allocation before bounds check
reject trailing garbage where required
```

## Не владеет

Нет:

```text
LinkFrame
MuxFrame
CircuitCell
crypto
socket I/O
business semantics
serde
```

Каждый протокольный crate владеет своими сообщениями и использует `chimera-codec`.

## Dependencies

```text
chimera-types
```

---

# 7. `chimera-device`

## Назначение

Абстракция локального data-plane endpoint.

Carrier соединяет Chimera с **другим Chimera node**.

Device соединяет Chimera с **локальным host/network stack**.

## Владеет

```text
PacketDevice
PacketRx
PacketTx

Packet
PacketBatch

DeviceMtu
DeviceCapabilities
DeviceStats
```

Контракт:

```text
receive local packets
send local packets
report MTU
report operational state
```

## Не владеет

Нет:

```text
TUN implementation
OS routes
NAT
firewall
routing policy
overlay routing
```

## Dependencies

```text
chimera-types
```

---

# 8. `chimera-crypto`

## Назначение

Абстрактный криптографический контракт Chimera.

Ни сеть, ни identity layer не должны знать конкретную crypto-library.

## Владеет

```text
CryptoProvider

Hash
Kdf
Kem
Signature
Aead
Csprng

PublicKey
SecretKeyHandle
SharedSecret
TrafficKey
Nonce
SignatureBytes

Transcript
KeySchedule
HybridSecret
```

Также:

```text
sign
verify

encapsulate
decapsulate

seal
open

derive
hash
```

## Криптографическая модель

MVP обязан поддерживать crypto-agility.

Default handshake profile предполагается hybrid:

```text
classical KEX
+
post-quantum KEM
        ↓
combined secret
        ↓
HKDF
        ↓
traffic keys
```

Например:

```text
X25519 + ML-KEM
```

Identity signatures также должны допускать hybrid profile:

```text
classical signature + ML-DSA
```

Конкретные suites принадлежат `chimera-crypto-default`.

## Не владеет

Нет:

```text
identity trust
certificates
network frames
keystore
filesystem
TLS
QUIC
```

## Dependencies

```text
chimera-types
```

---

# 9. `chimera-crypto-default`

## Назначение

Единственная стандартная production crypto implementation MVP.

`chimera-crypto` определяет контракт.

`chimera-crypto-default` его реализует.

## Владеет

Production implementations для:

```text
hash
KDF
CSPRNG
classical KEX
ML-KEM
classical signatures
ML-DSA
AEAD
```

Также владеет:

```text
DefaultCryptoProvider
DefaultHandshakeSuite
DefaultIdentitySuite
DefaultTrafficSuite
```

## Правило

Protocol identifiers определены явно.

Замена underlying library не должна менять wire semantics.

## Не владеет

Нет:

```text
identity policy
key persistence
handshake state machine
networking
```

## Dependencies

```text
chimera-types
chimera-crypto
```

Внешние cryptographic libraries выбираются отдельно и pin'ятся по версии.

---

# 10. `chimera-identity`

## Назначение

Определяет, **кто является кем**.

Crypto отвечает на вопрос:

> подпись математически корректна?

Identity отвечает:

> кому принадлежит ключ и чему он имеет право служить доказательством?

## Владеет

Три класса identity:

```text
User
Node
Service
```

Основные сущности:

```text
UserIdentity
NodeIdentity
ServiceIdentity

UserPublicIdentity
NodePublicIdentity
ServicePublicIdentity

NodeCertificate
ServiceCertificate

IdentityBundle
IdentityProof
IdentityAssertion

KeyId
KeyRole
KeyEpoch

TrustAnchor
TrustSet

IdentityManager
IdentitySigner
IdentityVerifier
```

## Identity hierarchy

Базовая модель:

```text
User root identity
      │
      ├── signs Node certificate
      │
      └── signs Service delegation
```

User root key не обязан находиться online постоянно.

Node использует собственный operational key.

## Идентификаторы

```text
UserId    = H(canonical user public identity)
NodeId    = H(canonical node public identity)
ServiceId = H(canonical service public identity)
```

IP/DNS в derivation не входят.

## Не владеет

Нет:

```text
filesystem
encrypted key files
SSH
OIDC
ZITADEL
routing
authorization policy
network connection
```

## Dependencies

```text
chimera-types
chimera-crypto
chimera-codec
```

---

# 11. `chimera-policy`

## Назначение

Чистый deterministic authorization engine.

Отвечает только:

```text
ALLOW
DENY
```

для известного контекста.

## Владеет

```text
Principal
AnonymousPrincipal

Action
Resource

Policy
Rule
Matcher
Decision

AdmissionContext
RouteContext
ServiceContext
```

Типовые решения:

```text
may node connect?
may user use exit?
may node relay?
may route cross peer?
may principal access service?
```

## Важный инвариант

Policy:

```text
pure input → deterministic decision
```

Она не выполняет I/O.

## Не владеет

Нет:

```text
OIDC
database
network calls
config parsing
route calculation
connection management
```

## Dependencies

```text
chimera-types
chimera-identity
```

---

# 12. `chimera-carrier`

## Назначение

Абстракция физической доставки Chimera Link между двумя endpoints.

Carrier ничего не знает о VPN, routing или user identity.

## Владеет

```text
Carrier
CarrierFactory

CarrierListener
CarrierDialer

CarrierConnection

CarrierEndpoint
CarrierCapabilities

CarrierError
CarrierStats
```

Connection предоставляет минимальные transport primitives:

```text
reliable record channel

optional:
datagram channel
```

Capability model:

```text
RELIABLE
DATAGRAM
SERVER_LISTEN
CLIENT_DIAL
MIGRATION
MULTIPATH
```

## Carrier не шифрует Chimera payload

Carrier может использовать TLS/QUIC собственным протоколом.

Но overlay security всегда обеспечивается `chimera-link`.

Это сознательное double-envelope separation.

## Не владеет

Нет:

```text
Chimera identity
peer authorization
routing
streams
circuit
VPN packet semantics
```

## Dependencies

```text
chimera-types
```

---

# 13. `chimera-peer`

## Назначение

Control-plane модель известного удалённого node.

Identity говорит **кто это**.

Peer говорит **как его сейчас можно достигнуть**.

## Владеет

```text
PeerRecord
VerifiedPeerRecord

ReachabilitySet
Reachability

PeerCapabilities

PeerHealth
EndpointHealth

PeerSnapshot
PeerTable

PeerSource
```

Пример:

```text
NodeId
 ├── QUIC endpoint A
 ├── H3 endpoint B
 ├── HTTPS endpoint C
 └── capabilities
```

## Инвариант

```text
NodeId != endpoint
```

Endpoint может исчезнуть.

Peer остаётся тем же peer.

## `PeerSource`

Абстракция получения peer records.

MVP implementations:

```text
static configuration
runtime learned peers
```

DHT/directory/gossip могут быть добавлены позже отдельными providers без изменения routing.

## Не владеет

Нет:

```text
live socket
Link
route calculation
connection establishment
DHT protocol
```

## Dependencies

```text
chimera-types
chimera-identity
chimera-carrier
```

---

# 14. `chimera-keystore`

## Назначение

Production local implementation `IdentityManager`.

## Владеет

```text
local key persistence
identity bundle persistence
atomic creation
atomic replacement
key rotation staging
key permissions
secret zeroization
public export
backup-safe public metadata
```

Default location:

```text
~/.chimera/
├── identity/
├── trust/
└── state/
```

На system deployment путь задаётся явно.

## CLI operations, которые обслуживает crate

```text
identity init
identity show
identity export-public
identity rotate
```

## Решение по `.ssh`

MVP **не использует `~/.ssh` как primary identity store**.

Причина:

```text
SSH key semantics != Chimera identity semantics
```

Позже возможен import/provider adapter.

## Решение по ZITADEL/OIDC

В MVP их нет.

Они являются внешними identity/control-plane adapters, а не частью network kernel.

## Не владеет

Нет:

```text
authorization policy
network login
OIDC
routing
sessions
```

## Dependencies

```text
chimera-types
chimera-crypto
chimera-identity
```

---

# 15. `chimera-carrier-quic`

## Назначение

Высокопроизводительный native carrier.

Основной carrier там, где network path не ограничивает QUIC.

## Владеет

```text
QUIC listener
QUIC dialer
QUIC connection adapter
datagram support
connection migration where available
transport metrics
```

## Цель

Минимальный overhead и максимальная throughput/latency efficiency.

## Не владеет

Нет:

```text
Chimera cryptography
peer identity
routing
censorship logic
```

## Dependencies

```text
chimera-types
chimera-carrier
```

---

# 16. `chimera-carrier-h3`

## Назначение

Стандартный HTTP/3 carrier.

Используется там, где требуется traffic compatibility с современным HTTP infrastructure.

## Владеет

Только H3-family transport:

```text
HTTP/3 session establishment
extended CONNECT
CONNECT-UDP / MASQUE mode
HTTP/3 datagrams where supported
```

## Принцип

Это не imitation protocol.

На wire находится валидный:

```text
QUIC
TLS
HTTP/3
```

## Не владеет

Нет:

```text
fake TLS
custom TLS fingerprint engine
Chimera routing
identity
```

## Dependencies

```text
chimera-types
chimera-carrier
```

---

# 17. `chimera-carrier-https`

## Назначение

TCP/443 fallback carrier.

Он существует отдельно от H3 по фундаментальной причине:

```text
QUIC/H3 = UDP
HTTPS   = TCP
```

Блокировка UDP не должна уничтожать все carrier'ы одновременно.

## Владеет

```text
TLS 1.3
HTTP/2
extended CONNECT / equivalent standards-based tunnel
reliable record transport
```

## Свойство

Это transport diversity не только на application layer, но и:

```text
UDP ↔ TCP
```

## Не владеет

Нет:

```text
Chimera crypto
routing
identity
VPN semantics
```

## Dependencies

```text
chimera-types
chimera-carrier
```

---

# 18. `chimera-link`

## Назначение

Создаёт один защищённый authenticated hop:

```text
Node A ⇄ Node B
```

поверх произвольного `CarrierConnection`.

Это главный security boundary сети.

## Владеет

```text
Link
LinkHandle

LinkHandshake
HandshakeState

LinkFrame

LinkState
LinkCloseReason

SessionKeys
TrafficKeyEpoch

RekeyPolicy
ReplayWindow

LinkLimits
LinkStats
```

## Handshake

Handshake выполняет:

```text
protocol negotiation
crypto-suite negotiation
hybrid key establishment
node authentication
transcript binding
session-key derivation
anti-replay
key confirmation
```

Результат:

```text
AuthenticatedLink {
    local_node,
    remote_node,
    negotiated_suite,
    traffic_keys,
    carrier,
}
```

## Link знает

```text
local NodeId
remote NodeId
carrier properties
session crypto
```

## Link не знает

```text
UserId consuming traffic
destination IP
route
circuit
service
VPN
```

## Dependencies

```text
chimera-types
chimera-codec
chimera-crypto
chimera-identity
chimera-carrier
```

---

# 19. `chimera-mux`

## Назначение

Мультиплексирование логических flows поверх одного Link.

## Владеет

Два primitives:

```text
Stream
DatagramFlow
```

### Stream

```text
ordered
reliable
flow-controlled
```

### DatagramFlow

```text
message-oriented
no retransmission requirement
suitable for tunneled IP/UDP-like traffic
```

Также:

```text
Mux
MuxHandle

MuxFrame

StreamId
FlowId

FlowControl
PriorityClass

Open
Close
Reset

MuxLimits
MuxStats
```

## Важный инвариант

Потеря одного Stream не должна блокировать независимые Streams.

Datagram traffic не должен искусственно превращаться в reliable byte stream, если carrier предоставляет datagrams.

## Не владеет

Нет:

```text
route selection
peer discovery
onion encryption
policy
TUN
```

## Dependencies

```text
chimera-types
chimera-codec
chimera-link
```

---

# 20. `chimera-route`

## Назначение

Чистый route planner.

Берёт immutable snapshot сети и выдаёт план.

Не открывает соединения.

## Владеет

```text
TopologySnapshot

RouteQuery
RouteConstraint

Hop
RoutePlan

RouteMetric
RouteScore

RoutePlanner
RouteError
```

Route может быть:

```text
Direct
Relay
MultiHop
Exit
Service-bound
```

## Input

```text
source
destination intent
peer snapshot
capabilities
health
policy constraints
route constraints
```

## Output

```text
RoutePlan
```

Пример:

```text
A → B → C → Exit D
```

## Инвариант

Route planner:

```text
pure snapshot → deterministic plan
```

при одинаковом tie-break seed.

## Не владеет

Нет:

```text
socket
Link
Circuit
packet forwarding
peer discovery
```

## Dependencies

```text
chimera-types
chimera-peer
chimera-policy
```

---

# 21. `chimera-circuit`

## Назначение

Реализует multi-hop logical path поверх существующих Mux sessions.

```text
A → B → C → D
```

## Владеет

```text
Circuit
CircuitHandle

CircuitBuilder
CircuitState

CircuitCell

RelayHop
RelayState

OnionKey
HopKeys

Extend
Extended
Destroy

CircuitLimits
CircuitStats
```

## Circuit construction

Input:

```text
RoutePlan
```

Process:

```text
establish hop 1
derive hop material
extend hop 2
derive hop material
...
```

## Data cells

Каждый relay получает только данные, необходимые для своей hop semantics.

Circuit crate предоставляет **механизм** multi-hop forwarding.

Он сам по себе не обещает полную Tor-equivalent anonymity.

Traffic-analysis resistance — отдельная security property, которая должна
