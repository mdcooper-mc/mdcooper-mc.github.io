+++
title = "Swaps Pricing Engine (MVP)"
description = "A production-ready C# library for pricing total return equity swaps, wrapping a COM-based pricing engine behind idiomatic .NET domain objects, a fluent builder API, and a layered validation framework."
tags = ["Finance", "C#", "Architecture", "Derivatives"]
+++

The Swaps Pricing Engine is a production-ready C# library that prices total return equity swaps against a proprietary
COM-based pricing engine. Rather than exposing the engine's unmanaged handle-based interfaces directly, the library
introduces a thin but complete domain model that maps naturally onto how traders and risk managers think about equity
swap contracts and positions. The result is a system that is easy to use correctly and difficult to use incorrectly —
enforcing market conventions through the type system, guiding consumers through a fluent API, and validating inputs
through a layered rule framework before anything reaches the pricing engine.

The MVP achieved an **81.6% pass rate across 500 live production positions**, covering same-currency and
cross-currency swaps, long and short positions, and a range of underlying equity types including stocks, ETFs, and
indices.

---

## The Problem

Equity swap pricing engines built on COM expose a complex, callback-driven programming model. Consumers must allocate
and manage opaque instrument handles, marshal data structures between managed and unmanaged memory, register callback
interfaces to receive results, and carefully manage resource lifetime to avoid leaks. This low-level surface area
makes integration error-prone and test coverage impractical.

At the same time, equity swaps carry a model ambiguity that trips up nearly every first implementation: the
distinction between contract notional and signed position exposure. An ISDA confirmation always expresses notional as
a positive number. Direction — which party pays which leg — is stated in legal language, not in the sign of a number.
Dealer systems translate this into a positive contract notional combined with a signed quantity, where positive means
long equity (receive equity returns, pay financing) and negative means short (pay equity returns, receive financing).
Encoding direction into the notional itself violates both ISDA conventions and the pricing engine's database
constraints, which enforce `Notional > 0` at the SQL schema level.

The library solves both problems: it hides the COM complexity behind idiomatic .NET, and it enforces the correct
contract/position separation through its type model.

---

## Domain Model

The library represents swap economics through two immutable records that mirror real-world market practice.

`SwapContract` encapsulates all static contract parameters agreed between counterparties — the underlying equity,
currencies, rate, spread, notional, day count convention, accrual adjustment, start and maturity dates, and initial
fixings. Notional is always positive. The contract has no directionality; it is the symmetric agreement both parties
reference.

`ContractPosition` links a contract to a signed quantity. A positive quantity represents long equity exposure — the
holder receives equity returns and pays floating. A negative quantity represents short equity exposure — the holder
pays equity returns and receives financing income. Position-level exposure is simply `Quantity × Notional`, making
aggregation and risk attribution straightforward.

`ContractMarketData` carries the market inputs required to price a contract: spot prices for the underlying, FX spots
for currency conversion, and historical equity and FX fixings for accrual calculations. It references the contract
directly, keeping market data co-located with the instrument definition it belongs to.

All three records implement `IDeterministic`, a marker interface that guarantees stable hash codes across the
application lifetime. This enables them to be safely used as keys in concurrent dictionaries and cached collections
without unexpected equality failures.

---

## Architecture

The system is organised into four layers that separate concerns cleanly.

The **domain objects** layer (`Data/Objects/`) defines `SwapContract`, `ContractPosition`, `ContractMarketData`,
`Underlying`, `Spot`, `Fx`, and `Fixing` as immutable C# records. These types carry no pricing logic and have no
dependency on the pricing engine. They are pure data.

The **data store** layer (`Data/Store.cs`) manages four in-memory stores that mirror the pricing engine's internal
partitions: instrument store, market data store, position store, and historical store. The `DataStore` record
routes `IDataItem` objects to the correct store by origin, emitting `BeforeDataSave` and `AfterDataSave` events so
consumers can observe every write. A `RemoveAll` method cleans up all items associated with a given session UUID,
preventing data leakage between pricing calls.

The **validation** layer (`Data/Objects/Validation/`) defines small, focused rule classes that each check one
constraint. Rules are registered with `ValidationProvider` and can be applied individually or all at once via
`ValidateAndThrow`. The `NotionalValidationRule` is representative: it throws a `ValidationException` with a
contextual message if `Notional <= 0`. Over 25 validation rules cover the contract, market data, and position
types, catching invalid inputs before they reach the pricing engine.

The **pricing API** (`API.cs`) exposes the engine through a type-state fluent builder. Interfaces partition the
valid transitions: `ICanSetDataStore → ICanSetReferenceData → ICanPrice`. This means the compiler prevents calling
`Price()` before providing a data store, a contract, market data, a position, and a pricing date. There is no way
to skip a required step.

---

## The Fluent Builder

The `EquitySwapPricer` implements all three builder interfaces and progresses through an internal `State` enum
(`None → Initialised → HasDataStore → ReadyToPrice`). Each method validates the current state before executing,
throwing a descriptive exception if called out of sequence.

`Init()` is a thread-safe singleton initialiser. It validates the presence of configuration and licence files, loads
required unmanaged assemblies, and initialises the COM interop layer exactly once using a double-checked lock. All
subsequent pricers share the same initialised state.

`With(DataStore)` attaches a data store to the session. `With(ReferenceData)` dispatches on the runtime type of the
argument — `SwapContract`, `ContractPosition`, or `ContractMarketData` — routing each to the correct data item
provider. `With(DateTime)` sets the pricing date and triggers context creation after verifying all three data objects
have been provided.

`Price(currency)` retrieves the registered position from the position store, calls `CalculatePortfolio` on the
pricing context, wraps the result in a `Calculation` record, and fires `OnPrice` or `OnPriceError` events
depending on outcome. `CleanUp()` removes all session data and resets state to `HasDataStore`, allowing the same
pricer instance to be reused for a subsequent position without re-initialising.

---

## Validation and Market Conventions

The validation framework enforces the contract/position model at runtime with explicit, actionable error messages.
Key rules include:

- Notional must be strictly positive — reflects the ISDA convention and the pricing engine's SQL `CHECK (Notional > 0)` constraint.
- Quantity must be non-zero — a zero quantity represents no position and has no economic meaning.
- Start date must precede maturity date — catches transposed dates early.
- Initial equity fixing must be positive — zero or negative fixings produce undefined accrual calculations.
- Required market data must be present — spot prices and FX rates for the contract's currencies must exist before pricing.

Validation runs before any data reaches the pricing engine, so failures surface as typed `ValidationException`
instances with context, not as opaque COM errors or silent incorrect results.

---

## Pricing Flow

A complete pricing call follows five steps. First, the consumer constructs the domain objects and calls
`ValidationProvider.ValidateAndThrow` on each. Second, the fluent builder registers the contract and position
as `IDataItem` objects in the instrument and position stores. Third, `With(DateTime)` creates a market data context
in the pricing engine, populating it from the market data store. Fourth, `Price(currency)` calls
`CalculatePortfolio`, which runs the total return swap pricer against the assembled context. Fifth, the `Calculation`
result exposes typed measure accessors — `Get<PV>()`, `Get<Delta>()`, `Get<Theta>()` — so consumers retrieve only
the values they need without parsing unstructured output.

The `Calculation` record carries a `Success()` method that returns false if the pricing engine reported errors,
allowing the consumer to branch cleanly without catching exceptions for expected failure modes.

---

## Cross-Currency Support

When the swap currency differs from the underlying equity currency, the engine operates in composite style. The
`IsCompoStyle` derived property is computed by comparing `SwapCurrency` against `Underlying.StockCurrency`. When
they differ, the market data must include both an equity FX spot and historical FX fixings for the currency pair.
The pricing engine applies FX adjustments transparently during valuation, converting equity leg cashflows into the
swap currency at each fixing date.

Self-referencing FX pairs — `JPY/JPY = 1.0`, `GBP/GBP = 1.0` — must be provided for same-currency swaps. The
pricing engine does not default these internally; omitting them produces a missing-data error at pricing time rather
than a silent wrong result.

---

## Data Architecture and Object Identity

The pricing engine uses a hierarchical object model: Book → Account → Position → Instrument, with market data
supplied through a separate context. Each object is identified by a composite key of `Name` and `AsOf` date, with
`Name` limited to 64 characters by the underlying SQL schema.

The library generates session-scoped names by combining a UUID prefix with the position identifier. Instrument names
embed the position UUID so that positions priced concurrently never collide in the shared instrument store. This
design enables thread-safe parallel pricing at the cost of creating unique instrument instances per position —
which matches the pricing engine's own data model, where each position holds its own instrument instance by design.

An analysis of 56,678 production JSON files confirmed that this intentional duplication carries no meaningful
redundancy: every position is unique by quantity, counterparty, and terms. An 84% space reduction is achievable
through JSON.NET metadata removal and GZIP compression without touching the object model.

---

## Skills

**Languages & Frameworks**
C# | .NET | COM Interop | log4net | SQLite

**Financial Domain**
Equity Swaps | Total Return Swaps | ISDA Conventions | Derivative Pricing | Position Economics | Delta / Theta

**Architecture & Design**
Type-State Builder Pattern | Domain-Driven Design | Immutable Records | Interface Segregation | Fluent API Design

**Validation**
Rule-Based Validation Framework | Layered Validation | Fail-Fast Validation | Typed Exceptions

**Concurrency & Safety**
Thread-Safe Initialisation | Double-Checked Locking | Session-Scoped Identifiers | Concurrent Data Stores

**Testing & Quality**
25+ Validation Rules | 500-Position Test Suite | 81.6% Pass Rate | Deterministic Data Structures

