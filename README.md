# Swarm public statistics verification project

This local controlled Git repository contains public, agent-authored source and no private input. It has no third-party dependencies or network calls.

The baseline exposes `Mean([]float64) (float64, error)`, preserves its input and rejects empty or non-finite input. Finite sums use Go's exact rational arithmetic to avoid intermediate overflow.

The approved next feature is `Median([]float64) (float64, error)`. It must preserve input order, support odd and even lengths, negative and duplicate values, reject empty input with `ErrEmptyInput`, reject non-finite values with `ErrNonFiniteInput`, and avoid overflow for two equal large finite middle values. The existing `Mean` contract remains covered.

Fixed verification commands are `go test ./...`, `go build ./...`, and `git diff --check`. Two actual native sessions will produce and review the feature through explicit context handoff. Native file and command tools remain closed; controlled application of reviewed changes and the fixed commands belong to the application side.
