---
title: "Kubeconfig files for Multiple Kubernetes Clusters" # Title of the blog post.
date: 2022-11-04T05:51:55-05:00 # Date of post creation.
summary: "Using a single kubeconfig file to interact with multiple Kubernetes clusters" # Description used for search engine.
codeMaxLines: 15 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
thumbnail: "/images/kubeconfig.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/kubeconfig.png" # Designate a separate image for social media sharing.
toc: true
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - Kubernetes  
---

## Background

If you're working with Kubernetes at all, you may be needing
to interact with more than one cluster - maybe different
clusters for development and production, maybe different
applications, or whatever.

Each cluster probably has a `kubeconfig` file that you
can use to interact with *that cluster*, but changing context
based on different files is a bit cumbersome. Options include
changing environment variable for each context change or
using the `--kubeconfig` command line option with most
commands.

With a little extra one-time work for each cluster you
have, changing context can be a snap.

## Anatomy of a Kubeconfig File

Each cluster you have will often be able to provide you
with a `kubeconfig` file, and they look something like this:

```yaml
---
apiVersion: "v1"
kind: "Config"

clusters:
- name: "erik-test"
  cluster:
    server: "https://some-url:6443"
    certificate-authority-data: "LS0tLS1CRUdJTiBDRVJUSU...."

contexts:
- name: "erik-test-admin@erik-test"
  context:
    cluster: "erik-test"
    user: "erik-test-admin"

users:
- name: "erik-test-admin"
  user:
    client-certificate-data: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS...."
    client-key-data: "LS0tLS1CRUdJTiBSU0EgUFJJVkFU...."

current-context: "erik-test-admin@erik-test"
```

The `kubeconfig` file above represents a ***single cluster***,
but the same overall structure can be used by a single `kubeconfig` file
to store context for multiple clusters.

- **clusters:** contains one or more `cluster` item
- **contexts:** contains one or more `context` item
- **users:** contains one or more `user` item
- **current-context:** contains exactly one reference to the `name`
property of a `context`

## Modifying the Default Kubeconfig File

The default `kubeconfig` file is `~/.kube/config` - and that's what
should contain your collection of clusters. Modifying that is pretty
easy - do that once for each cluster you have, and switching between
contexts is pretty straight-forward.

In the above example, you would add this content to the `clusters` node:

```yaml
- name: "erik-test"
  cluster:
    server: "https://some-url:6443"
    certificate-authority-data: "LS0tLS1CRUdJTiBDRVJUSU...."
```

Then add this to the `contexts` node:

```yaml
- name: "erik-test-admin@erik-test"
  context:
    cluster: "erik-test"
    user: "erik-test-admin"
```

Then add this to the users node:

```yaml
- name: "erik-test-admin"
  user:
    client-certificate-data: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS...."
    client-key-data: "LS0tLS1CRUdJTiBSU0EgUFJJVkFU...."
```

### Give a Friendly Name to your Context

As a bonus note, the `name` of each `context` is whatever is easy
for you to understand, so examples of `name` values you might use
instead of `erik-test-admin@erik-test` are:

- `dev-local-identity`
- `prod-aws-identity`
- `dev-identity@admin`

Including some info about the application, location, envrironment,
and possibly user are possible suggestions - just use a `name` value
that is meaningful to you.

## Changing Contexts

Once you've got the changes made to your default `kubeconfig` file,
you can use the following command to change the context:

```bash
kubectl config use-context <name-of-context>
```

So in the example from the `kubeconfig` content above, the following command would work:

```bash
kubectl config use-context erik-test-admin@erik-test
```

The last argument is the `name` of the context you are switching to,
and that's why choosing a value that's meaningful to you is important.

To get a list of the contexts you have available, use the following:

```bash
kubectl config get-contexts
```

## Include Cluster Name in Prompt

As a further aid in keeping track of your current context, you
may want to include the current context (maybe even with the
namespace!!) in your prompt if you are using something like
[Oh-My-Posh](https://ohmyposh.dev/docs/segments/kubectl).

I've written [a post covering how to do that]({{< ref "2021/03/add-k8s-cluster-to-windows-terminal" >}}).

In case you're curious, though, this is the current theme I'm
using for Oh-My-Posh:

![oh-my-posh theme](/images/dracula-k8s-posh.png)

The json is here:

```json
{
  "$schema": "https://raw.githubusercontent.com/JanDeDobbeleer/oh-my-posh/main/themes/schema.json",
  "blocks": [
    {
      "alignment": "left",
      "segments": [
        {
          "background": "#6272a4",
          "foreground": "#ffffff",
          "leading_diamond": "\ue0b6",
    "trailing_diamond": "\ue0b0", 
          "style": "diamond",          
          "type": "os"          
        },
        {
          "background": "#bd93f9",
          "foreground": "#ffffff",
          "powerline_symbol": "\ue0b0",    
          "style": "powerline",          
    "properties": {
     "style": "folder"    
    },
          "type": "path"          
        },
        {
          "background": "#ffb86c",
          "foreground": "#ffffff",
          "powerline_symbol": "\ue0b0",
          "properties": {
            "branch_icon": "",
            "fetch_stash_count": true,
            "fetch_status": false,
            "fetch_upstream_icon": true
          },
          "style": "powerline",
          "template": " \u279c ({{ .UpstreamIcon }}{{ .HEAD }}{{ if gt .StashCount 0 }} \uf692 {{ .StashCount }}{{ end }}) ",
          "type": "git"
        },        
        {
          "background": "#ff79c6",
          "foreground": "#ffffff",
    "powerline_symbol": "\ue0b0",    
          "style": "powerline",
          
          "type": "kubectl",
          "properties": {    
            "prefix": " \uF1D1 "
          }
        }
      ],
      "type": "prompt"
    }
  ],
  "final_space": true,
  "version": 2
}
```
