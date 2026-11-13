# Spec Delta

## Purpose

Turns a Cursor catalog entry into the model parameters Cursor receives and the context window DSH is told about, including tier defaults, per-model overrides, and recovery from parameters Cursor rejects.

## ADDED Requirements

### Requirement: Largest context tier by default

The route SHALL request the largest context tier a model offers unless a smaller tier is selected explicitly.

#### Scenario: Model offering several tiers
- **WHEN** a request runs on a model whose catalog entry offers `300k` and `1m`, and no override selects a tier
- **THEN** the parameters sent to Cursor request `1m`

#### Scenario: Model offering a single tier
- **WHEN** a request runs on a model whose catalog entry offers exactly one context tier
- **THEN** the parameters sent to Cursor request that tier

#### Scenario: Model offering no context parameter
- **WHEN** a request runs on a model whose catalog entry has no context parameter
- **THEN** the request is sent without a context parameter and the route does not invent a tier

### Requirement: Explicit tier selection wins

When a tier is selected explicitly, by configuration or environment, the route SHALL request exactly that tier rather than the largest offered one.

#### Scenario: Explicit tier matches an offered value
- **WHEN** the configured tier is `300k` and the model offers `300k` and `1m`
- **THEN** the parameters sent to Cursor request `300k`

#### Scenario: Explicit minimum tier
- **WHEN** the configured tier is `min` and the model offers `300k` and `1m`
- **THEN** the parameters sent to Cursor request `300k`

#### Scenario: Explicit tier is not offered
- **WHEN** the configured tier is a value the model does not offer
- **THEN** the route falls back to the largest offered tier and never sends the unoffered value

### Requirement: Per-model tier override

The route SHALL honor a tier recorded for one model without applying it to any other model.

#### Scenario: Override applies to its own model only
- **WHEN** a tier is recorded for one model and a request runs on a second model
- **THEN** the second model requests its own largest offered tier

#### Scenario: Override takes precedence over the global selection
- **WHEN** a tier is recorded for a model and the global selection asks for a different tier
- **THEN** the recorded tier is requested for that model

### Requirement: Rejected tier steps down and is remembered

When Cursor rejects a request because the model parameters are invalid, the route SHALL retry once with the next smaller offered tier, and SHALL remember the working tier for that model so later requests start there.

#### Scenario: Rejection steps down one tier
- **WHEN** a first request on a model is rejected with an invalid-parameters error and the model offers a smaller tier
- **THEN** the route retries at the next smaller offered tier and the turn continues

#### Scenario: Working tier survives a restart
- **WHEN** a working tier was learned for a model and the host process restarts
- **THEN** the first request on that model uses the learned tier without paying a rejected attempt

#### Scenario: Smallest tier is already in use
- **WHEN** a request is rejected and no smaller offered tier exists
- **THEN** the route surfaces the error instead of retrying

#### Scenario: Rejection after output was emitted
- **WHEN** a run produced visible output before failing with an invalid-parameters error
- **THEN** the route surfaces the error and does not replay the turn at a smaller tier

### Requirement: Reported context window matches the requested tier

The context window reported to DSH for a model SHALL be the token size of the tier the route requests, and SHALL NOT be a fixed default when the catalog states a tier.

#### Scenario: Tier is known from the catalog
- **WHEN** DSH asks for a model whose catalog entry offers `300k` and `1m`
- **THEN** the reported context window is that of the tier the route would request

#### Scenario: Tier is unknown
- **WHEN** DSH asks for a model whose catalog entry states no readable context tier
- **THEN** the route reports no context window rather than an unrelated fixed number

### Requirement: Max Mode variants are accepted

When a catalog variant that matches the requested parameters requires Cursor Max Mode, the route SHALL send the request in a form Cursor accepts instead of failing with an invalid-parameters error.

#### Scenario: Model whose largest tier requires Max Mode
- **WHEN** a request selects the tier of a variant marked as requiring Max Mode
- **THEN** the run completes instead of failing with an invalid-parameters error

#### Scenario: Model whose tier does not require Max Mode
- **WHEN** a request selects a tier that runs outside Max Mode
- **THEN** the run completes and requests no more than the selected tier
