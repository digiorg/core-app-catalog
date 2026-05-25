# size-resolver

KCL function that resolves a T-shirt size (`S`, `M`, `L`) from an `AppClaim`'s `spec.size`
field into concrete resource values consumed by Crossplane Compositions.

## Status

Stub — implementation follows with C-4 / C-5.

## Purpose

Compositions need concrete numbers (CPU millicores, memory MiB, storage GiB, replica counts,
connection pool sizes). Rather than duplicating the mapping in every Composition, `size-resolver`
centralises the logic in a single KCL function and injects the resolved values into the composite
resource status, where Compositions pick them up via patches.

## Input

`spec.size` on the `AppClaim` / `Application` composite:

```yaml
spec:
  size: S   # one of: S | M | L
```

## Output

The function writes resolved values into the composite resource (e.g. `status.atProvider`):

| Field | Description |
|-------|-------------|
| `cpuRequest` | CPU request for application containers |
| `cpuLimit` | CPU limit for application containers |
| `memoryRequest` | Memory request for application containers |
| `memoryLimit` | Memory limit for application containers |
| `storage` | PVC / CNPG cluster storage size |
| `replicas` | Desired replica count |
| `connectionPoolSize` | CNPG PgBouncer max connections |

## T-shirt Size Table

| Size | CPU Request | CPU Limit | Mem Request | Mem Limit | Storage | Replicas | Pool |
|------|------------|-----------|-------------|-----------|---------|----------|------|
| S | 50m | 200m | 64Mi | 256Mi | 1Gi | 1 | 5 |
| M | 200m | 500m | 256Mi | 512Mi | 5Gi | 2 | 20 |
| L | 500m | 2000m | 512Mi | 2Gi | 20Gi | 3 | 50 |

## Usage in a Composition

```yaml
- step: resolve-size
  functionRef:
    name: function-kcl
  input:
    apiVersion: krm.kcl.dev/v1alpha1
    kind: KCLRun
    spec:
      source: |
        # See functions/size-resolver/ for full source
```
