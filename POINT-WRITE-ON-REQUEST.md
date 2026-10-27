# writeOnRequest patch

This fork adds an optional per-point `writeOnRequest` marker to Haxall 4.0.7.
It allows an explicit request to reissue an unchanged winning write through
the existing point-write observation path. Unmarked points retain their
existing behavior.

## Purpose

Haxall normally suppresses duplicate writes when a priority slot already
contains the same value and `who`. Updating the PLC readback does not change
that priority slot. This is useful for control applications which repeatedly
request the same output, but two cases benefit from an explicit reissue:

- An external application sends an API heartbeat by writing `true`. The PLC
  consumes it and resets its register to `false`. The application's next
  request must be able to write `true` again. If requests stop, the PLC
  watchdog must expire rather than receive replacement heartbeats generated
  automatically by Haxall.
- Haxall acts as an HMI for a setting also changed by another HMI. After
  Haxall writes 10 and the other HMI changes the PLC value to 5, a new request
  for 10 should be able to reassert that setting without first releasing the
  priority slot or changing priorities.

The marker does not rewrite automatically when readback differs. Reissues
require another explicit write request. `writeMaxTime` provides periodic
rewrites, while `writeOnStart` and `writeOnOpen` provide lifecycle rewrites;
they do not tie each heartbeat to a new request from the external application.

## Behavior

Configure `writeOnRequest` on the writable point, not on its tuning record.
The existing write functions and API signatures remain unchanged. For example,
with the marker on `pt`, repeated `pointWrite(pt, true, 16)` calls can reissue
the effective write observation.

A reissue is eligible when the requested value is non-null, equals the current
effective value, and the request is at the winning priority. This includes
requests with an unchanged `who` and requests which only change `who`.
Eligible reissues reach both normal effective-write subscribers and
`obsAllWrites` subscribers.

Priority-array changes remain separate from observation eligibility. An exact
duplicate does not repeat slot updates, persistence or effective-value
calculation. Lower-priority requests cannot reissue the winning output, and
null releases retain their existing behavior. Authorization, priority
arbitration, timed overrides and the steady-state startup rules are unchanged.

Connector scheduling, conversion, coalescing and tuning are also unchanged,
including `writeMinTime`, `writeMaxTime`, `writeOnStart` and `writeOnOpen`.
An observation is not a request-specific device acknowledgement. Concurrent
requests may still be coalesced or delayed by existing connector behavior;
the marker does not guarantee one device command per API call.

## Implementation

The production changes are limited to:

- `src/ext/hxPoint/fan/WriteRec.fan`: preserve priority-array updates and pass
  effective changes or eligible reissues to the existing observation path.
- `src/ext/hxPoint/fan/WriteMgr.fan`: name the observation flag
  `notifyEffective` to cover both effective changes and reissues.
- `src/xeto/hx.point/specs.xeto`: define the optional point marker.

The observable documentation and point guidance describe the new exception.
No connector implementation or public write API was changed.

## Initial validation

Local validation passed on 27 October 2026 against the current fork changes
based on official Haxall 4.0.7. Builds and tests ran in a disposable Docker
container without changing the live application containers or their databases.
Additional regression helpers were compiled in the disposable source copy,
not added to the repository's test suite.

The repository build scripts successfully compiled `hx`, `hxPoint`, `hxConn`,
`testXeto`, `testHx` and `hxModbus`. Xeto builds passed for `hx.point`,
`hx.modbus`, `hx.test` and `hx.test.conn`.

The regression run passed 66 test methods with 15,517 assertions. Coverage
included marked and unmarked duplicates, Boolean values, changed `who`,
losing priorities, null releases, marker changes, conversion, startup,
timed override renewal and expiry, observation counts, authenticated API
writes, connector coalescing and minimum/maximum write intervals.

Two live integration tests used a local Mod_RSsim simulator on
`localhost:502`, unit 1, accessed from Docker through `host.docker.internal`:

- Holding register offset 0 (`40001`), unsigned 16-bit: write 10, independently
  change the register to 5, and confirm an unmarked repeat remains suppressed
  for five seconds. Adding the marker alone caused no write. Two subsequent
  requests for 10 each produced a native FC06 write and acknowledgement,
  including the request made after readback already matched 10. The test
  passed 51 assertions and restored the original value of 0.
- Coil offset 0 (`00001`): write `true`, independently reset the coil to
  `false`, and confirm the same unmarked suppression. With the marker,
  repeated `true` requests produced fresh FC05 writes and acknowledgements.
  A change to `false` followed by another `false` request also produced two
  writes, confirming that `false` qualifies as non-null. The test passed
  62 assertions and restored the original value of `false`.

The live tests counted native requests and acknowledgements, checked readback,
and serialized requests to avoid conflating reissues with queue coalescing.
The disposable connectors were disabled and their priority slots released
before restoration; restored values were verified by readback.

The first coil attempt timed out on its baseline read, before any coil write.
The simulator had accumulated 13 `CLOSE_WAIT` connections and stopped answering
register reads too. After its listener was restarted, the test helper reused
one control connection per test and the coil-only rerun passed. No production
change was needed for that test-environment issue.

Across the successful regression and live runs, 68 test methods passed with
15,630 assertions. The simulator checks demonstrate the native Modbus path;
they do not establish delivery guarantees for every connector.

## Planned pilot use

The patch will be run in the THEO pilot application to evaluate explicit
API-initiated watchdog heartbeats and deliberate reassertion of settings.
The initial validation above is local testing, not completed pilot operation;
the test session did not deploy the patch to the live pilot containers.

Pilot validation will need to confirm the behavior through the actual
application and connectors, with their configured write intervals and
coalescing behavior. In particular, stopping the external application's
heartbeat requests must still allow the PLC watchdog to expire. The marker
does not disable independently configured periodic or lifecycle rewrites.
