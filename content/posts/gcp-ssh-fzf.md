+++ 
draft = false
date = 2024-01-05T10:15:10Z
title = "Using gcloud with fzf to interactively SSH to GCP VMs"
description = ""
slug = ""
authors = []
tags = ["gcloud", "ssh", "iap", "fzf", "cloud"]
categories = []
externalLink = ""
series = []
+++

The `gcloud compute ssh` [command](https://cloud.google.com/sdk/gcloud/reference/compute/ssh) is a wrapper around SSH that is used to
SSH to VMs in GCP.

[fzf](https://github.com/junegunn/fzf) is a command-line fuzzy finder, as the project site states it is an interactive UNIX filter for command-line that can be used with any list, files, etc.

The example function below (which can be added to your bash or zsh profile) can be used to interactively select the VM to login to using `gcloud compute ssh` or SCP a file to a instance.  

```bash
function gcp_ssh(){
    proj_filter="$1"
    PROJECT=$(gcloud projects list --format="value(projectId)" --filter="name~$proj_filter" | fzf --tac)
    INSTANCE_INFO=$(gcloud compute instances list --project $PROJECT | fzf --tac)
    INSTANCE=$(echo $INSTANCE_INFO | awk '{print $1}')
    ZONE=$(echo $INSTANCE_INFO | awk '{print $2}')
    gcloud compute ssh "$INSTANCE" --zone "$ZONE" --project "$PROJECT"
}

function gcp_scp(){
    proj_filter="$1"
	filepath="$2"

    PROJECT=$(gcloud projects list --format="value(projectId)" --filter="name~$proj_filter" | fzf --tac)
    INSTANCE_INFO=$(gcloud compute instances list --project $PROJECT | fzf --tac)
    INSTANCE=$(echo $INSTANCE_INFO | awk '{print $1}')
    ZONE=$(echo $INSTANCE_INFO | awk '{print $2}')

    gcloud compute scp "$filepath"  "$INSTANCE":~/ --zone "$ZONE" --project "$PROJECT"
}
```

You can then use the command `gcp_ssh` on its own or `gcp_ssh project` by providing the full or partial GCP project name.
