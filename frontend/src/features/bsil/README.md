# BSIL Frontend Surfaces

BSIL components render reviewer-facing truth states and governed confidence
bands. They must not convert backend `degraded`, `invalid`, `unknown` or
`unavailable` states into successful UI states.

Semantic and behavioral outputs must stay observational. Do not use
accusatory wording or imply adjudicated misconduct from a confidence band.
