# Spec Delta

## Purpose

Makes a starting turn observable instead of silent, and prepares the model runtime ahead of the first request so a starting agent does not wait on per-process and per-workspace setup.

## ADDED Requirements

### Requirement: A starting turn emits visible progress

The route SHALL emit a visible assistant block before any Cursor model output arrives, so a turn is never silent while the run is being created or is waiting on its first token.

#### Scenario: Cold start
- **WHEN** a turn starts on a model runtime that has not been used before and the model has produced no output yet
- **THEN** DSH receives a visible assistant block before the first Cursor chunk arrives

#### Scenario: Progress gives way to real output
- **WHEN** the first Cursor text or thinking chunk arrives after progress was emitted
- **THEN** the progress block ends and the model output is streamed as usual

#### Scenario: Recovered start
- **WHEN** a turn is retried internally after a rejected parameter set
- **THEN** the turn again becomes observable before its next attempt begins

#### Scenario: Progress is not counted as model output
- **WHEN** progress was emitted and the run then fails
- **THEN** the failure is reported as an error rather than being treated as a completed turn with content

### Requirement: A starting turn does not wait indefinitely in silence

While a turn is waiting for the model runtime, the route SHALL make that wait observable or bounded, so a stalled run does not present as an unresponsive agent.

#### Scenario: Run runtime retries internally
- **WHEN** the model runtime retries a transport failure internally before producing output
- **THEN** the turn remains visibly in progress for the whole wait

#### Scenario: Wait exceeds the bound
- **WHEN** no output arrives before the configured bound
- **THEN** the route ends the attempt with an error outside DSH's replay set instead of staying silent

### Requirement: Runtime preparation targets the real working directory

The route SHALL prepare the model runtime for the working directory a session will actually use, and SHALL keep the preparation alive while that directory is in use.

#### Scenario: Preparation matches the session directory
- **WHEN** the route prepares the runtime before a session's first turn
- **THEN** the preparation targets the directory that session's first turn will use

#### Scenario: Preparation covers a large workspace
- **WHEN** the route prepares the runtime for a workspace that is slow to scan
- **THEN** the first turn on that workspace does not repeat the scan

#### Scenario: Preparation is released
- **WHEN** the route shuts down
- **THEN** the runtime preparation it holds is released instead of leaked

### Requirement: Preparation starts without waiting for a catalog

The route SHALL start runtime preparation when the plugin loads, when a key is available, rather than waiting for a model catalog request.

#### Scenario: Key available at load
- **WHEN** the plugin loads and a Cursor key is resolvable
- **THEN** runtime preparation starts without any user turn and without a catalog listing request

#### Scenario: No key at load
- **WHEN** the plugin loads with no resolvable Cursor key
- **THEN** the route stays idle and starts preparation when a key first becomes available

#### Scenario: Preparation is disabled by configuration
- **WHEN** preparation is disabled by configuration or environment
- **THEN** the route skips it and turns still run

#### Scenario: Preparation fails
- **WHEN** runtime preparation fails
- **THEN** the failure is contained and turns still run
