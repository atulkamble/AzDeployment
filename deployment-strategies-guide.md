# Deployment Strategies: Practical Guide

This guide compares blue-green, canary, rolling, feature flag, traffic routing, zero-downtime, and Azure App Service slot swap techniques. The strategies can be combined; for example, a slot swap is one way to implement a blue-green release, while traffic routing can gradually shift users to a canary.

## 1. Blue-Green Deployment

Maintain two production-like environments:

- **Blue** serves current production traffic.
- **Green** runs the new release and is validated before traffic moves.

### Release flow

1. Deploy the new version to the inactive environment.
2. Run health checks and smoke tests against its private or staging URL.
3. Move the production route or load balancer to Green.
4. Monitor errors, latency, and business metrics.
5. If needed, route traffic back to Blue while it remains available.
6. After the observation period, update Blue for the next release.

**Advantages:** fast cutover, straightforward rollback, production-like validation.

**Trade-offs:** temporarily needs capacity for both environments; database changes must remain compatible with both versions.

## 2. Canary Deployment

Release to a small share of users first, then increase exposure after reviewing telemetry.

### Example progression

`5% → 10% → 25% → 50% → 100%`

At each step, compare error rate, latency, availability, and key business outcomes with the stable version. Pause or roll back if a defined threshold is exceeded.

**Use when:** you need to limit the impact of defects and can identify or route a controlled user segment.

**Trade-offs:** requires reliable traffic controls, monitoring, and enough traffic for useful comparisons.

## 3. Rolling Deployment

Replace instances in batches while the rest continue serving requests.

### Release flow

1. Confirm remaining instances have enough capacity.
2. Remove a small batch from the load balancer.
3. Update and health-check that batch.
4. Return healthy instances to service.
5. Repeat until all instances run the new version.

Set a minimum healthy capacity and stop automatically when health checks fail. Keep old and new versions compatible during the rollout.

**Use when:** the platform supports instance batches and spare capacity is limited.

**Trade-offs:** old and new versions coexist; rollback may require another rollout.

## 4. Feature Flags

Deploy code with a feature disabled, then enable it for selected users or conditions through configuration.

```text
if feature_flags.new_checkout.enabled_for(user):
    show_new_checkout()
else:
    show_existing_checkout()
```

### Safe usage

- Default new features to off.
- Enable for internal users, then a small audience, then expand.
- Record flag changes and monitor the feature separately.
- Remove temporary flags after the rollout.
- Keep authorization and security checks server-side; a flag is not an access control boundary.

Feature flags separate **deployment** (code is present) from **release** (users can access it). They do not replace testing or a rollback plan.

## 5. Traffic Routing

Control which version receives requests using a router such as a load balancer, ingress controller, API gateway, DNS, or application platform.

Routing can be based on:

- Percentage of requests (weighted routing)
- User or tenant identity (sticky or cohort routing)
- Region, header, cookie, or path
- Health and availability of a target

Use short-lived routing changes where possible, confirm session behavior, and account for DNS caching when DNS is involved. Routing directs requests; it does not by itself deploy or validate an application.

## 6. Zero-Downtime Deployment

Zero downtime is an outcome, not a single deployment method. It requires the application, infrastructure, and data changes to allow service to continue throughout release.

### Checklist

- At least one healthy instance remains available during updates.
- Readiness checks prevent traffic from reaching unready instances.
- Graceful shutdown lets in-flight requests finish.
- Capacity supports the rollout while instances are unavailable.
- Database changes use an expand-and-contract sequence:
  1. Add backward-compatible schema elements.
  2. Deploy code that can work with old and new schema.
  3. Migrate data if needed.
  4. Remove obsolete schema only after old code is gone.
- External dependencies and background jobs tolerate mixed versions.
- Rollback and monitoring procedures are ready.

A deployment can avoid infrastructure downtime while users still experience errors if schema changes, sessions, or dependencies are incompatible.

## 7. Azure App Service Slot Swap

Deployment slots provide separate live app instances with slot-specific hostnames. A common pattern is to deploy to a `staging` slot, validate it, and swap it with `production`.

### Azure CLI example

```bash
# Set values for your app
RG="myResourceGroup"
APP="myapp-name"

# Create a staging slot (run once)
az webapp deployment slot create \
  --resource-group "$RG" \
  --name "$APP" \
  --slot staging

# Deploy a ZIP package to staging
az webapp deploy \
  --resource-group "$RG" \
  --name "$APP" \
  --slot staging \
  --src-path ./app.zip \
  --type zip

# Check staging before swapping
curl -fsS "https://${APP}-staging.azurewebsites.net/health"

# Swap staging into production
az webapp deployment slot swap \
  --resource-group "$RG" \
  --name "$APP" \
  --slot staging \
  --target-slot production

# Verify production
curl -fsS "https://${APP}.azurewebsites.net/health"
```

Use the app's actual default hostname or custom domain. Configure environment-specific settings, connection strings, and secrets correctly; mark settings that must stay with a slot as deployment-slot settings. Review swap behavior and warm-up/health-check configuration before using it for production. A reverse swap can restore the prior app version, but data changes may not be reversible.

## 8. Selecting a Strategy

| Situation | Good starting choice | Key condition |
|---|---|---|
| Need a fast switch and quick app rollback | Blue-green or slot swap | Both versions can run with the same data safely |
| Want to limit exposure to a small audience | Canary | Traffic can be segmented and monitored |
| Many interchangeable instances, constrained extra capacity | Rolling | Capacity remains sufficient during batches |
| Need to deploy code before making it visible | Feature flags | Flags are controlled, monitored, and later removed |
| Need gradual or cohort-based exposure | Traffic routing plus canary | Router supports the required targeting |
| Strict availability requirement | Rolling or blue-green plus zero-downtime practices | App, database, and dependencies support mixed versions |

### Decision questions

1. Can old and new versions use the same database schema at the same time?
2. How quickly must rollback restore service?
3. Can traffic be divided by percentage, user, tenant, or region?
4. Is there enough capacity to run duplicate environments or spare instances?
5. Which health and business metrics will stop the rollout?
6. Do background workers, sessions, caches, and external APIs support mixed versions?

## Recommended release controls

- Define health checks and success thresholds before deployment.
- Use a staged rollout for high-impact changes.
- Keep the prior version available until the new release is stable.
- Make database migrations backward-compatible and separately planned.
- Automate rollback triggers where practical, and document the manual path.
- Record who released what, when, and to which environment.
