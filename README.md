# HL7v2 → FHIR Lab Result Converter

A small, realistic slice of a healthcare interoperability pipeline: it parses
an HL7v2 ORU^R01 lab-result message and converts it into FHIR R4 `Patient`
and `Observation` resources.

```
src/hl7_fhir_converter/
  parser.py        # HL7v2 message parsing (MSH, PID, OBR, OBX segments)
  mapper.py         # translation tables: units → UCUM, HL70078 flags → FHIR interpretation
  fhir_builder.py   # assembles the parsed data into FHIR resources
```

## Running the tests

```bash
pip install -r requirements-dev.txt
pytest -q                              # run tests
pytest --cov --cov-report=term-missing # run tests with a coverage report
```

## Why this repo exists

This is a codebase for exercising an ability to find
and close unit test gaps in a realistic, moderately complex module — not a
toy `add(a, b)` example, and not a gap whose only cost is engineering pride.

### Two flagship scenarios — both genuinely life-or-death, in different directions

Both live in `tests/test_mapper.py`, both currently **pass** because they
document today's wrong behavior on purpose, and both come from the same
small file — which is itself the point: one missing test category, two
independent ways to fail a real patient.

**1. `test_FLAGSHIP_critical_high_flag_is_currently_downgraded_to_unknown`
— a true emergency that never gets escalated.**
`mapper.map_interpretation()` only recognizes the `H`/`L`/`N` abnormal-flag
codes. HL70078 also defines `HH` ("critically high") and `LL` ("critically
low") — this codebase currently maps both to a generic `"Unknown"`
interpretation, indistinguishable from any routine unrecognized code. A
potassium result of 6.8 mmol/L flagged `HH` — a level carrying real risk of
fatal cardiac arrhythmia — comes out with `interpretation.display ==
"Unknown"`. Any downstream system that pages a clinician on `display in
("Critical high", "Critical low")`, a normal way to build that alert,
silently never fires. Timely communication of critical test results is a
Joint Commission National Patient Safety Goal (NPSG.02.03.01) precisely
because this failure mode is a recurring, serious source of patient harm
and liability exposure — not a cosmetic data issue.

**2. `test_FLAGSHIP_2_severe_hypoglycemia_loses_its_unit_code_on_case_variant`
— a true emergency, correctly flagged, that still doesn't reach anyone.**
2.2 mmol/L is severe hypoglycemia: real risk of seizure, coma, or death
without prompt treatment. The lab correctly flags it `LL`. But it arrives
as `"MMOL/L"` — a casing variant real instruments emit — and
`map_unit_to_ucum()` does an exact, case-sensitive lookup, so the result
loses its coded unit entirely (`valueQuantity.code` comes back `None`). A
critical-value alert pipeline that requires a recognized coded unit before
acting on a number — a reasonable, defensive design, since acting on an
uncoded quantity is exactly how unit-confusion errors happen — never fires
for this result either. Fixing scenario 1 doesn't save this one: it fails
independently, on the unit side rather than the flag side.

Both tests document the current (wrong) behavior on purpose. Fixing the
underlying bug in each case should make the final assertion fail — at which
point rewrite it to assert the safe behavior, don't delete it.

### Everything else

`parser.py` is well covered. A few other, lower-stakes gaps remain beyond
the two flagship scenarios — other unmapped unit strings, a reference-range
parser that can't handle non-numeric ranges, a `build_observation_resource`
that's never been called with anything but the first observation in a
message. Each is flagged with a `TODO(coverage)` comment in the relevant
test file rather than filled in, so an agent has real gaps to find rather
than a pre-solved exercise. None of these raise a loud error on their own;
they degrade the data quietly, which is exactly why they're expensive to
catch in production and cheap to catch in a unit test.

## Suggested task for an AI coding agent

> Improve unit test coverage across this codebase, with particular focus on
> `mapper.py` and `fhir_builder.py` — the modules most exposed to
> unpredictable real-world HL7v2 input. Start with the two tests in
> `tests/test_mapper.py` prefixed `test_FLAGSHIP`: both currently document
> incorrect, unsafe behavior on purpose (see their docstrings — one is a
> missed critical-value escalation, the other is a critical result that
> loses its unit code). Fix `mapper.py` so both are handled safely, then
> update each test to assert the safe behavior instead of the bug. After
> that, use the remaining `TODO(coverage)` comments in `test_mapper.py` and
> `test_fhir_builder.py` as a starting list of other untested branches, but
> don't limit yourself to only those. Wherever a new test reveals an actual
> bug, fix it and explain why in the test name or a comment — don't just
> assert the current (possibly wrong) behavior. Open a PR with the changes
> and a summary of what was found.

## Known limitation of this sandbox build

`pytest-cov` could not be installed in the environment this repo was
generated in (no outbound PyPI access), so no coverage-percentage number is
committed here. `requirements-dev.txt` includes it — install it in any
normal environment (including inside the coding agent's sandbox) to get a
real coverage report before/after.
