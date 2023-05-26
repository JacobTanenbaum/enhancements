---
title: upgrades-with-ovn-ic
authors:
  - "@ricky-rav"
reviewers:
  - "@trozet"
  - "@tsurya"
approvers:
  - "@trozet"
api-approvers:
  - "None"
creation-date: 2023-05-26
last-updated: 2023-05-26
tracking-link:
  - https://issues.redhat.com/browse/SDN-3905
---

# Upgrades and rollbacks with OVN interconnect

**This is a very very initial draft**

## Summary
Allow a 4.13 cluster to smoothly upgrade to 4.14, which features OVN IC multizone. 
Allow a 4.14 multizone cluster to roll back to 4.14 1-zone if needed and also to switch back to multizone.
## Motivation
The default in 4.14 will be to have multizone IC (1 node per zone). We need to support:
- upgrades from 4.13, where OVNK has no zone support, going through an intermediate step with a 1-zone configuration
- configurations with one global zone, equivalent to today's architecture
- possibly, configurations with more than one node per zone (nice-to-have)

Anything different than the default multizone configuration needs to be captured by a ConfigMap, which will include the necessary details to:
- describe the desired non-default zone mode 
- distinguish between upgrades and non-default (final) configurations

For instance, in order to have a permanent 1-zone configuration, a user should push:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ovn-interconnect-configuration
  namespace: openshift-ovn-kubernetes
data:
  zone-mode: singlezone
  definitive: yes
```

Additionally, we can also envisage fields to describe a configuration with multiple zones and more than one node per zone. This can be left for future work, if requested.

### User Stories


### Goals


### Non-Goals


## Proposal


## Design Details

### upgrade 4.13 -> 4.14 1-zone -> 4.14 multizone
- When the user starts the upgrade from 4.13 to 4.14, CNO knows that the FromVersion (4.13) has no zone support, so in its upgrade path it will start by pushing a ConfigMap that will be used to track the intermediate upgrade step of 1-zone IC, needed before we reach the final goal of multizone IC:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ovn-interconnect-configuration
  namespace: openshift-ovn-kubernetes
data:
  zone-mode: singlezone
  definitive: no
```

- CNO will then rollout 1-zone YAMLs: first ovnk node, then ovnk master

- We have now: 4.14 1-zone ovnk, IC configmap with "definitive: no". *We enter CNO upgrade path thanks to the configmap with "definitive: no": can we?*
  + CNO rolls out multizone YAMLs. *How does CNO know whether 1-zone YAMLs have all rolled out and we're now in 1-zone IC? By inspecting nodes annotations and tracking rollout progress*
  + CNO nukes old OVN DBs
  + once that is done, the Config Map is also removed

Apart from the removal of the configmap, here the steps are the same as in the cases described below, where there's no upgrade and we want to change the IC configuration. Code should be shared and here it should be called from within the upgrade path. CNO should update its operator status only when the final step of switching to multizone has completed.


### 4.13 -> 4.14 1-zone

- The user must specify this non-default configuration by pushing the following configmap before starting the upgrade:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ovn-interconnect-configuration
  namespace: openshift-ovn-kubernetes
data:
  zone-mode: singlezone
  definitive: yes
```


- CNO will see the configmap with `definitive: yes` and won't yet enter the upgrade path
- The user will start the upgrade
- CNO enters the upgrade path:
  + it sees `FromVersion=4.13` (no zone support) and finds the configmap already in the API server; no action on the configmap is needed;
  + it rolls out 1-zone YAMLs, at the end of which the upgrade is done
  + the configmap is kept


### 4.14 multizone -> 4.14 1-zone
CNO needs a way to know the _current_ zone configuration
+ It can read the zone annotation from all nodes and store it in cache.
+ For multizone, it will store `zone-mode=multizone` when it sees:
   - node1: zone1
   - node2: zone2
   - ...
   - nodeN: zoneN
+ For 1-zone, it will store `zone-mode=singlezone` when it sees:
  - node1: global-zone
  - node2: global-zone
  - ...
  - nodeN: global-zone

+ Any other case will result in `zone-mode=undefined` until we support more configurations.

- When node annotations change, `zone-mode` must be re-computed. => requires listening on node changes
- When the IC configmap is created or modified, `zone-mode` must be re-computed  => requires listening on configmap changes 

Upon changes to the IC-configuration configmap (add or update or delete), CNO compares the current `zone-mode` with the desired one:
  + easy case: current zone mode is either multizone or 1-zone and is different from the desired configuration in the configmap. CNO triggers the rollout of the desired YAMLs.
  + harder case: current zone mode is `undefined`, desired zone mode is either `multizone` or `1-zone`. Once CNO knows that there's no ongoing rollout, it should try to converge to the desired zone mode by triggering the rollout of the appriopriate YAMLs.

- the configmap is kept.


### 4.14 1-zone -> 4.14 multizone
- Similarly to the previous case, we can be permanently in 4.14 1-zone only with a configmap that demanded so.
- if now the user removes this configmap, the desired zone mode is multizone (default), so CNO will compare that against the current zone mode and will trigger a rollout with multizone YAMLs


### 4.14 fresh install
By default, openshift 4.14 will start in multizone mode, that is with one node per zone. 
Any non-default configuration must be specified through the same configmap shown above, thanks to which CNO will render then right YAML files.

QUESTION: can we push a configmap before installation time?? Maybe... but this is not supported.

A post-install change in IC configuration will have to be done through a configmap and will follow the steps outlined above.

## Additional resources
https://docs.google.com/presentation/d/1pWjEHkWfJ5NDo9jpEfE-CRPrHyzLaaw2So4bbUXJZXQ/edit#slide=id.p

## Test Plan
