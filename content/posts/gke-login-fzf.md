+++ 
draft = false
date = 2024-01-05T10:51:41Z
title = "Authenticate with GKE and AKS clusters interactively with fzf"
description = ""
slug = ""
authors = []
tags = ["google","cloud", "gke", "azure", "aks", "fzf"]
categories = []
externalLink = ""
series = []
+++

[fzf](https://github.com/junegunn/fzf) is a command-line fuzzy finder, as the project site states it is an interactive UNIX filter for command-line that can be used with any list, files, etc.

Following on from our other posts on using fzf, here is an example of a function that can be used to to authenticate/login to a GKE cluster interactively:

```bash
function gcp_gke_login(){
  PROJECT=$(gcloud projects list --format="value(projectId)" | fzf --tac)
  RESULT=$(gcloud container clusters list --project=$PROJECT --format="value(name,location)" | fzf --tac)
  CLUSTER_NAME=$(echo "$RESULT" | awk '{print $1}')
  REGION=$(echo "$RESULT" | awk '{print $2}')
  KUBECONFIG_CLUSTER=gke_"$PROJECT"_"$REGION"_"$CLUSTER_NAME"
  echo "Manipulating kubeconfig entry using cluster: $KUBECONFIG_CLUSTER"
  gcloud config set component_manager/disable_update_check true
  gcloud container clusters get-credentials --project="${PROJECT}" --region="${REGION}" "${CLUSTER_NAME}"
}
```

We can use fzf with the Azure CLI in a similar fashion to login to an AKS cluster:

```bash
function az_aks_login(){
  sub=$(az account list -o table --query "[].{Subscription:name}"|fzf --tac)
  cluster=$(az aks list -o table --subscription $sub | fzf --tac)
  cn=$(echo $cluster | awk '{print $1}')
  rg=$(echo $cluster | awk '{print $3}')
  az aks get-credentials -g $rg -n $cn --subscription $sub
  echo "kubectl config current-context"
  kubectl config current-context
}
```
