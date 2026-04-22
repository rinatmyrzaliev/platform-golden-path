# Versioning Policy — Platform Library Chart

## Semver contract

The library chart follows [Semantic Versioning](https://semver.org/).
Consumer charts pin a specific version in their `Chart.yaml` dependency
block and upgrade on their own schedule.

```
MAJOR.MINOR.PATCH

MAJOR — breaking changes (consumers must update values or templates)
MINOR — new features, backward compatible
PATCH — bug fixes, backward compatible
```

## What counts as a breaking change (MAJOR bump)

- Adding a new `required` field (e.g. a new mandatory label)
- Renaming or removing a values key that consumers currently use
- Changing the structure of rendered output in a way that causes resource
  recreation (e.g. changing matchLabels, renaming a template)
- Removing a template (e.g. dropping `platform.pdb`)
- Changing the default behavior of an existing feature (e.g. PDB default
  changing from on to off)

## What counts as a new feature (MINOR bump)

- Adding a new optional template (e.g. `platform.hpa`)
- Adding a new optional values key with a sensible default
- Adding a new label that doesn't affect selectors
- Supporting a new field in an existing template (e.g. adding `annotations`
  support to the Service template)

## What counts as a bug fix (PATCH bump)

- Fixing incorrect indentation in rendered YAML
- Fixing a nil pointer error on optional fields
- Correcting a label value format issue
- Updating documentation

## Version history

| Version | Ticket  | Change |
|---------|---------|--------|
| 0.1.0   | PLAT-001 | Initial library: Deployment template with probes, security context, labels |
| 0.2.0   | PLAT-002 | Added Service and Ingress templates |
| 0.3.0   | PLAT-003 | Added PDB, topology spread, on_call_channel required field |
| 0.4.0   | PLAT-004 | Added ExternalSecret template, envFrom wiring |
| 1.0.0   | PLAT-005 | Stable release. All templates mature, ArgoCD integration verified |

## How consumers upgrade

1. Check the library changelog for breaking changes
2. Update `Chart.yaml` dependency version:
   ```yaml
   dependencies:
     - name: platform-library
       version: "1.1.0"
       repository: "file://../../charts/platform-library"
   ```
3. Run `helm dependency update` to pull the new version
4. Run `helm template` and diff against previous output
5. Commit and push — ArgoCD syncs the change

## Deprecation policy

When a feature will be removed in the next MAJOR version:

1. MINOR release adds a deprecation warning (rendered as a Helm `fail` message
   with `--set allowDeprecated=true` to suppress)
2. Deprecation notice stays for at least one MINOR release cycle
3. MAJOR release removes the feature

This gives teams at least one version window to migrate before anything breaks.