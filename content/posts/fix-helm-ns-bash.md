+++
title = "How to Fix a Helm Namespace [PART 1]"
date = 2019-10-14
[taxonomies]
tags = ["encoding", "understandings", "kubernetes", "helm", "protobuf", "bash"]
+++

## Problem Statement

In [Helm][helm] it is possible to template the namespace of kubernetes resources from values instead of using the information in the .Release object.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: "namespace from .Values"
  namespace: {{ .Values.namespace }}
```
vs.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: "namespace from .Release"
  namespace: {{ .Release.Namespace }}
```

##### NOTE: The second example above is actually default, and the same as not specifying namespace at all.

It turns out overridding the namespace like this is not a good idea. This results in [tiller][tiller] storing a namespace in its view of the world that does not match reality. Further, there is not actually any way to later update tiller so that what is stored in its configmap matches reality.

Well, not any easy way...

## Environment Setup

#### In order to actually try any of this code the following is assumed:
- access to an existing kubernetes cluster
- ability to use kubectl
- ability to use helm cli

#### Further in order to decode the protobuf in the tiller configmaps:
1. Get the .proto definitions from helm

  - `go get k8s.io/helm`

2. Install the protoc cli tool

  - `sudo dnf install protobuf-devel`

## Find the Tiller ConfigMap for the Chart

The first thing we need to do is **programmatically** find the configmap for the chart deployment we are trying to fix.

```bash
# This is the directory with the helm .proto files
HELM_PROTO_DIR=$GOPATH/src/k8s.io/helm/_proto

# We are not using any:
# argument parsing
# usage instructions
# help text
# IMHO, once you need those things,
# you should probably no longer be using bash
chart_name=$1
to_namespace=$2

# Create some tempfiles to use for intermediary steps and debugging
decoded_cm=$(mktemp)
fixed_cm=$(mktemp)
fixed_base64=$(mktemp)

pushd $HELM_PROTO_DIR

# Get all of the tiller configmaps that are actually deployed
jp='{.items[?(@.metadata.labels.STATUS=="DEPLOYED")].metadata.name}'
tiller_cm_list=$(kubectl get cm -n kube-system -o jsonpath=$jp)

# Find the configmap for the particular chart we want to modify
# There is likely a better way to do this
for cm in $tiller_cm_list; do
  if (echo $cm | grep -E ^$chart_name); then
    tiller_cm=$cm
  fi
done
```
## Decode the Tiller ConfigMap

The tiller configmaps have a `.data.release` key which stores everything. This is protobuf encoded binary which is gzip'd and then base64 encoded.

So we just need to decode it:

```bash
# Get .data.release field from Tiller CM and decode it
kubectl get cm \
  -n kube-system $tiller_cm -o jsonpath='{.data.release}' |
  base64 -d |
  gunzip |
  protoc --decode  \
    hapi.release.Release \
    hapi/release/release.proto > $decoded_cm
```

## Modify the release
With the configmap release field decoded, we can now modify the namespace with a simple `sed`
```bash
# Change namespace in decoded Tiller CM
# This is one of the more fragile parts of this method
sed 's/namespace: "default"/namespace: "'$to_namespace'"/' \
  < $decoded_cm > $fixed_cm
```

## Re-encode the Tiller ConfigMap
Then we can re-encode it and patch it back to the configmap
```bash
# Re-encode release to put back in Tiller CM
protoc --encode \
  hapi.release.Release \
  hapi/release/release.proto < $fixed_cm |
    gzip -c |
    base64 -w0 > $fixed_base64

# Patch .data.release field in Tiller CM with fixed namespace
kubectl patch cm -n kube-system $tiller_cm \
  -p '{"data":{"release":"'$(cat $fixed_base64)'"}}'
```
## Problems With This Approach

1. It is pretty dangerous to be manipulating the decoded protobuf with sed
2. The user interface of the script is pretty terrible
3. In order to just run the script it is required to have a checkout of the helm git repo
4. Operating on one chart at a time is pretty tedious... especially if you need to update ~50 charts on ~10 clusters

## How Can We Do Better

Next time we will look at how we can solve all of the above issues using Rust.

[helm]: https://github.com/helm/helm
[tiller]: https://helm.sh/docs/using_helm/#installing-tiller
