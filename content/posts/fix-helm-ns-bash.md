+++
title = "How to Fix a Helm Namespace Without Needing to Re-deploy"
date = 2019-10-06
[taxonomies]
tags = ["encoding", "understandings", "kubernetes", "helm", "protobuf", "bash"]
+++

### Problem Statement
### Environment Setup

assumes an existing k8s cluster, ability to use kubectl and helm CLI tools against it

go get k8s.io/helm
install protobuf cli tool

### Find the Tiller ConfigMap for the Chart

#!/bin/bash
set -eux
set -o pipefail

HELM_CHECKOUT=$HOME/go/src/k8s.io/helm

component=$1
to_namespace=$2
decoded_cm=$(mktemp)
fixed_cm=$(mktemp)
fixed_base64=$(mktemp)

pushd $HELM_CHECKOUT/_proto
# Get the latest Tiller CM for the component
set +x
tiller_cm_list=$(kubectl get cm -n kube-system -o jsonpath='{.items[?(@.metadata.labels.STATUS=="DEPLOYED")].metadata.name}')

for cm in $tiller_cm_list; do
  if (echo $cm | grep -E ^$component); then
    tiller_cm=$cm
  fi
done

echo "============================================================================================="
echo "============================================================================================="
echo "UPDATING CONFIGMAP: $tiller_cm"
echo "============================================================================================="
echo "HELM STATE BEFORE OPERATION:"
helm ls | grep -E ^$component
echo "============================================================================================="
echo "============================================================================================="

# Get .data.release field from Tiller CM and decode it
kubectl get cm -n kube-system $tiller_cm -o jsonpath='{.data.release}' |
    base64 -d |
    gunzip |
    protoc --decode  hapi.release.Release hapi/release/release.proto > $decoded_cm

# Change namespace in decoded Tiller CM
sed 's/namespace: "default"/namespace: "'$to_namespace'"/' < $decoded_cm > $fixed_cm

# Re-encode release to put back in Tiller CM
protoc --encode hapi.release.Release hapi/release/release.proto < $fixed_cm |
    gzip -c |
    base64 -w0 > $fixed_base64

# Patch .data.release field in Tiller CM with fixed namespace
kubectl patch cm -n kube-system $tiller_cm -p '{"data":{"release":"'$(cat $fixed_base64)'"}}'

echo "============================================================================================="
echo "============================================================================================="
echo "HELM STATE AFTER OPERATION:"
helm ls | grep -E ^$component
echo "============================================================================================="
echo "============================================================================================="


popd
### Decode the Tiller ConfigMap
### Modify and Re-encode the Tiller ConfigMap
### Problems With This Approach
### How Can We Do Better


Unlike future posts, this post will not have much in the way of practical content. Instead, I will layout my intentions for this blog in general, as well as what my plans are for the first few posts. If those posts are already written by the time you see this, it is probably a better use of your time to skip this one. At some point, I will likely just fold information contained here into the about page and delete this post.

### Purpose

I have spent most of my professional career being paid to work solely and directly on open source software[1]. However, recently I have changed jobs. While I really enjoy the technical challenges I am working on, I miss the feeling of giving back to the worldwide community of software developers. This blog is my method to scratch that particular itch.

### Topics

There are two topics that I find endlessly facinating, and that I feel I have **practical** understandings to share. These are software engineering and Buddhism. As far as the blog goes, I intend to only make posts about things in these categories which provide some immediate **practical** value[2].

### The Spark

I have been thinking of starting a blog for years, but the spark to actually do it comes from a recent interaction I had with the [Helm][helm] community on slack. I needed to update the namespace that [tiller][tiller] thought a bunch of helm charts were deployed in[3]. After much googling without success, I asked on the [helm slack][helm-slack] channel. It turns out there is not any tooling to do this, but with some guidance from some very helpful folks in that channel, I was able to figure it out.

### Next Few Posts

The next few posts will be about the iterations of my solution to this problem. The first post will be about my first implementation simply written in Bash. However, there is much to learn even in this simple implementation about how tiller[4] maintains its state through configmaps with highly encoded data[5]. This Bash implementation worked, and I was able to use it against some dev clusters successfully. However, it felt very fragile and required cloning helm in order to run it[6].

The next **N** posts will be about how we can improve on this working but imperfect solution[7]. This is a pattern I would like to try with future topics. My mantra is: "Can I do better?". So, I will posit possible answers to that at the end of each post, and the following post will be about implementing those improvements.

Beyond that, I am not sure exactly what I will write about. I have a few weeks to think about it ...

### Cadence

I am a big proponent of habits[8]. As such, I plan to make a post every week. This may also change[9], but that is the plan for now.

### Footnotes

[1] Thanks Shadowman[1a]!

[1a] Also, R.I.P. Shadowman.

[2] For some definition of practical.

[3] Not the actual k8s namespaces they were deployed in[3a]. Just the ones stored in tiller's configmaps.

[3a] That is not possible because k8s namespaces are immutable on a deployed resource.

[4] The value of this knowledge is fading with Helm 3 around the corner[4a], but there will likely still be Helm 2 deployments around for quite some time.

[4a] R.I.P. tiller

[5] base64 and gzip and protobuf... Oh my!

[6] for the .proto files

[7] **SPOILER ALERT** it involves rust.

[8] I meditate every day.

[9] Impermanence

[helm]: https://github.com/helm/helm
[tiller]: https://helm.sh/docs/using_helm/#installing-tiller
[helm-slack]: https://kubernetes.slack.com/archives/C0NH30761/p1569422690003400
