# ecommerce-kpt

A reusable kpt package for a simplified e-commerce deployment on Kubernetes.  
Variants are applied with `kubectl kustomize` — no manual file copying required.

## Services

| Service  | Image             | Role                        |
|----------|-------------------|-----------------------------|
| frontend | nginx:1.25-alpine | Mock storefront placeholder |
| catalog  | nginx:1.25-alpine | Product catalog             |
| checkout | nginx:1.25-alpine | Order / checkout            |

All configuration lives in a single `ecommerce-config` ConfigMap.  
Deployments and Services exist only in `base/`. Variants patch only what they own.

## Not Included

- Real application code or UI
- Database or persistence layer
- Payment processing
- Ingress / TLS
- Autoscaling (add HPA on top as needed)

---

## Prerequisites

```
kpt    >= v1.0
kubectl >= v1.26   (kustomize is built in since v1.14)
```

---

## Deploy Base (Default)

Base = boutique theme · en/USD · US sales tax · small profile

```bash
kpt pkg get https://github.com/YOUR_ORG/ecommerce-kpt/base@main ecommerce-base

kubectl kustomize ecommerce-base/ | kubectl apply -f -
```

---

## Apply a Variant

Each variant directory is a self-contained kustomization that references `base/` and patches only its own fields.

```bash
# Syntax
kubectl kustomize variants/<dimension>/<name>/ | kubectl apply -f -

# Examples
kubectl kustomize variants/theme/florist/          | kubectl apply -f -
kubectl kustomize variants/locale/inr/             | kubectl apply -f -
kubectl kustomize variants/tax/india-gst/          | kubectl apply -f -
kubectl kustomize variants/size/medium/            | kubectl apply -f -
```

### Combining variants

Variants are independent overlays. To combine two dimensions (e.g. florist theme + INR locale), create a composed overlay that references both patches:

```yaml
# overlays/florist-inr/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: ../../variants/theme/florist/configmap-patch.yaml
    target:
      kind: ConfigMap
      name: ecommerce-config
  - path: ../../variants/locale/inr/configmap-patch.yaml
    target:
      kind: ConfigMap
      name: ecommerce-config
```

```bash
kubectl kustomize overlays/florist-inr/ | kubectl apply -f -
```

---

## Variants Reference

### Theme

| Variant                        | `theme`         | `store_name`    |
|-------------------------------|-----------------|-----------------|
| base (default)                | boutique        | My Boutique     |
| variants/theme/florist        | florist         | Petal & Bloom   |
| variants/theme/car-accessories| car-accessories | AutoParts Pro   |

Patches: `theme`, `store_name` in `ecommerce-config`.

### Locale / Currency

| Variant              | `language` | `currency` | `currency_symbol` | `date_format` |
|----------------------|------------|------------|-------------------|---------------|
| base (default)       | en         | USD        | $                 | MM/DD/YYYY    |
| variants/locale/inr  | hi         | INR        | ₹                 | DD/MM/YYYY    |
| variants/locale/eur  | de         | EUR        | €                 | DD.MM.YYYY    |

Patches: `language`, `currency`, `currency_symbol`, `date_format` in `ecommerce-config`.

### Tax

| Variant                   | `tax_mode`    | `tax_rate` | `tax_label` | `tax_inclusive` |
|---------------------------|---------------|------------|-------------|-----------------|
| base (default)            | us_sales_tax  | 0.08       | Sales Tax   | false           |
| variants/tax/india-gst    | india_gst     | 0.18       | GST         | true            |
| variants/tax/eu-vat       | eu_vat        | 0.20       | VAT         | true            |

Patches: `tax_mode`, `tax_rate`, `tax_label`, `tax_inclusive` in `ecommerce-config`.

### Deployment Size

| Variant              | `profile` | Replicas | CPU limit (each) | Memory limit (each) |
|----------------------|-----------|----------|------------------|---------------------|
| base (default)       | small     | 1        | 100–200m         | 128–256Mi           |
| variants/size/medium | medium    | 2        | 300–400m         | 256–512Mi           |
| variants/size/large  | large     | 4        | 500–1000m        | 512Mi–1Gi           |

Patches: `profile` in `ecommerce-config`; `replicas` and `resources` in each Deployment.

---

## What Is Intentionally NOT Configurable

- Container images (edit `base/*-deployment.yaml` directly)
- Service ports
- Namespace name (hardcoded `ecommerce`)
- Readiness probe paths
- `CATALOG_URL` env var in checkout (hardcoded `http://catalog:8080`)

---

## Validation

### 1. Local render check (no cluster required)

Render a variant and inspect the output before applying:

```bash
kubectl kustomize variants/locale/inr/ | grep -A6 'name: ecommerce-config' | grep -E 'currency|language'
```

Expected output:
```
  currency: INR
  currency_symbol: ₹
  language: hi
```

### 2. Cluster state check (after apply)

```bash
kubectl get configmap ecommerce-config -n ecommerce \
  -o jsonpath='{range .data}{@}{"\n"}{end}'

kubectl get deployments -n ecommerce \
  -o custom-columns='NAME:.metadata.name,REPLICAS:.spec.replicas,READY:.status.readyReplicas'
```

For `variants/size/medium`, expected:
```
NAME       REPLICAS   READY
checkout   2          2
catalog    2          2
frontend   2          2
```
