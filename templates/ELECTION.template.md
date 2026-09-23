# ELECTION -- <unit> / round <n> / purpose ELECTION / pass <p>

## Open items register
(The running register of this election meeting: one line per open item, rewritten every pass, hashed with this file; empty at close.)
- <item id> (raised by <seat id>, pass <p>): <one-line state>

The frozen artifact of the election meeting. Verdicts never live in this file: they are in the append-only meeting log, whose closing record names this file's sha256. Fill every field. Only engine-specific fields absent by design in Standalone mode may be marked "inapplicable: <reason>"; missing required binding or agreement evidence blocks execution.

## Binding
- unit: <unit id>   round: <n>   planning closure: <the revision id of the plan's SAME PAGE>
- frozen roster: <seat id = model @ effort; ...>
- plan file: <path> sha256 <hash>
- contracts: <task id: path sha256 <hash>; ...>
- prior election superseded: <none | the revision id of the election this replaces, and why>

## Each seat's independent reading (one block per seat, written before merging)
### <seat id>
- capability chart: <the cells consulted, quoted with their calibration marks>
- SETUPS selector row (lane x class): <quoted verbatim>
- SETUPS selector-state row (last_probe / qualifying_since_probe / probe_due / next_axis): <quoted verbatim>
- PRIORS: <the rows consulted, quoted>
- pool readings: <pool: used / remaining; source; reading time; age; reset time; ...>
- measured vs projected: <what is measured for this task class; what is projected and from what>
- source-record locators: <file and line, or provider/runtime session and message/turn identifier, of each reading>

## Election, per task
### <task id>
- arm @ effort: <arm@effort>   setup: <S0 | S1 | S2 | S3 | S4>   agent count: <n>
- assignments, ownership, dependencies: <who does what; what depends on what>
- S0 justification: <the routing-record reason, or "not S0">
- cadence probe: <eligible yes/no; due yes/no; disposition: probe taken | exemption cited to <policy or scoped Owner override> | not due; outstanding debt preserved: <...>>
- fallback conditions: <what re-elects, on what evidence, to which ladder>
- rationale: <measured evidence first; projections labelled as such>
