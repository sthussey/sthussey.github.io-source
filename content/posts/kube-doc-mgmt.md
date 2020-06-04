---
title: "Kubernetes Document Management"
date: 2020-05-30T07:49:56-05:00
sidebar: right
authorbox: true
tags:
  - kubernetes
  - kustomize
  - helm
  - cue
---

Kubernetes' API is accessed by building YAML documents describing,
or declaring, your desired state of various Kubernetes resources.
Kubernetes then continually works to bring the actual state of the
resource in alignment with the desired state described in the documents.

For a small application deployed on Kubernetes, managing these resource
documents isn't difficult. But in an enterprise environment where
tens of applications are being deployed into multiple environments
managed by several teams, managing the documents describing all
of these deployments becomes a true chore. There are however several
tools to assist in this endeavor and this article considers each
and then compares how they fare in a few common use cases.


## A Simple Deployment

To compare how these tools are utilized, I prepared a simple deployment of a single
service into Kubernetes with a dedicated service account. This service is then tuned
to deploy into three different environments - development, test, and production.
Lastly we need a way to introduce CI into our system for policy compliance.

See the [Github repo](https://github.com/sthussey/doc-mgmt) for a reference.

First each tool needs to provide a method for a developer to provide a "package" for
deploying their code into an environment. Kubernetes is based on containers running
pre-built images, so the package isn't about the code itself. Rather the package is
about describing to Kubernetes the desired configuration of the service and its
supporting infrastructure

### Helm

[Helm](https://helm.sh) is less a configuration management system and more a "package"
manager for sets of Kubernetes resource definitions. I only include it here because
there are a lot of discussions of 'kustomize vs. helm', so I figured it should be 
included for context.

The base package is Helm's core functionality. You define templates 
in [Go's templating language](https://golang.org/pkg/text/template/)
(Helm v3 may support Lua at some point) that emit Kubernetes resource definitions.
The templates are driven by a set of configuration values that are described in one or
more YAML documents. The package is called a `chart` and each chart has a set of
default values for configuration. When the chart is deployed, the user can specify
one or more override documents that replace or augment the default configuration values.
There is currently no way to describe a policy with a range of allowed values for
configuration so that overrides can be validated.

First you can create a new chart.

```
 ~/helm$ helm create grass
```

The default chart is actually more complex than what we want to deal with, so you can
delete all the files in the `templates` directory aside from what you see below. Below
that are the default configuration values

Note in the deployment that the `replicas` field is defined by a template
expression sourcing the value from a variable `.Values.replicaCount`. This is
how the configuration defaults and overrides are injected into the resource
definitions.

```
 ~/helm/pkg$ ls -R grass
 grass:
 charts  Chart.yaml  templates  values.yaml
 
 grass/charts:
 
 grass/templates:
 deployment.yaml  serviceaccount.yam
 ~/helm/pkg$ cd grass
 ~/helm/pkg/grass$ cat templates/deployment.yaml
 ---
 apiVersion: apps/v1
 kind: Deployment
 metadata:
   name: crm-deployment
   labels:
     deployed-by: hand
 spec:
   selector:
     matchLabels:
       app: crm
   replicas: {{ .Values.replicaCount }}
   template:
     metadata:
       labels:
         app: crm
     spec:
       serviceAccountName: crm-serviceaccount
       containers:
         - name: crm-api
           image: grasscrm:1.2
           livenessProbe:
             httpGet:
               path: /
               port: https
           readinessProbe:
             httpGet:
               path: /
               port: https
 ...
 ~/helm/pkg/grass$ cat templates/serviceaccount.yaml
 ---
 apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: crm-serviceaccount
 ...
```
```
 ~/helm/pkg/grass$ cat values.yaml
 # Default values for grass.
 # This is a YAML-formatted file.
 # Declare variables to be passed into your templates.

 replicaCount: 1
```

We can now generate the resource definitions by applying
the default configuration values to the templates. This will
emit the YAML documents to be fed to Kubernetes. As can be
seen below the template variable `.Values.replicaCount` was
replaced with `1` as defined in the default configuration inside
of `values.yaml`.

```
 ~/helm/pkg$ helm template grass
 ---
 # Source: grass/templates/serviceaccount.yaml
 apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: crm-serviceaccount
 ...
 ---
 # Source: grass/templates/deployment.yaml
 apiVersion: apps/v1
 kind: Deployment
 metadata:
   name: crm-deployment
   labels:
     deployed-by: hand
 spec:
   selector:
     matchLabels:
       app: crm
   replicas: 1
   template:
     metadata:
       labels:
         app: crm
     spec:
       serviceAccountName: crm-serviceaccount
       containers:
         - name: crm-api
           image: grasscrm:1.2
           livenessProbe:
             httpGet:
               path: /
               port: https
           readinessProbe:
             httpGet:
               path: /
               port: https
 ...
```

### Kustomize

[kustomize](https://kustomize.io) is a template-free system for creating base and overlay
definitions for Kubernetes resources. It is Kubernetes specific.

In Kustomize, documents are built from a set of bases and overlays. The base would be basically equivalent
to a package as it would be the default resource definitions. Note there are no variables. Instead
configuration tuning is done by layered overrides being patched into the base resource definition.

Like Helm, Kustomize does not support much validation without custom scripting. There are some recipes
on integrating with separate utilities which we'll see later.

```
 ~/kustomize$ ls pkg
 deployment.yaml  kustomization.yaml  serviceaccount.yaml
 ~/kustomize$ cat pkg/kustomization.yaml pkg/deployment.yaml pkg/serviceaccount.yaml
 resources:
   - deployment.yaml
   - serviceaccount.yaml
 ---
 apiVersion: apps/v1
 kind: Deployment
 metadata:
   name: crm-deployment
   labels:
     deployed-by: hand
 spec:
   selector:
     matchLabels:
       app: crm
   replicas: 1
   template:
     metadata:
       labels:
         app: crm
     spec:
       serviceAccount: crm-serviceaccount
       containers:
       - name: crm-api
         image: grasscrm:1.2
         livenessProbe:
           httpGet:
             path: /
             port: https
         readinessProbe:
           httpGet:
             path: /
             port: https
 ...
 ---
 apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: crm-serviceaccount
 ...
```

And then we can `build` the deployment as defined in `kustomization.yaml`. 

```
 ~/kustomize$ kustomize build pkg/
 apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: crm-serviceaccount
 ---
 apiVersion: apps/v1
 kind: Deployment
 metadata:
   labels:
     deployed-by: hand
   name: crm-deployment
 spec:
   replicas: 1
   selector:
     matchLabels:
       app: crm
   template:
     metadata:
       labels:
         app: crm
     spec:
       containers:
       - image: grasscrm:1.2
         livenessProbe:
           httpGet:
             path: /
             port: https
         name: crm-api
         readinessProbe:
           httpGet:
             path: /
             port: https
       serviceAccount: crm-serviceaccount
```

### Cue

[Cue](https://cuelang.org) is both a tool and a language according to their docs. There is
a lot of deeper theory in their documentation, but it is similar to kustomize in that
it is a template-free system creating segmented resource definitions. It includes the
ability to define schemas and policy validation. It is not Kubernetes specific, and in
fact not even YAML specific. Still a very young project, but I think holds a lot of
promise for managing a complex document ecosystem.

Cue is similar to Kustomize in that the package is just the resource definition. The difference is
the definition is written in the Cue language and the base definition includes both base values as
well as schema definition to validate final outcomes including user configuration. The validation
rules can also be augmented with user definitions.

This time the `replicas` field has a different look - no variable, but something more than a
default value to be overriden. Cue does not support overrides. Instead the string `*1 | int` says
the field value must be either `1` or some integer. Obviously `1` is also an integer, but it is
listed explicitly because the `*` denotes it is the default. So `*1 | int` both defines a default value
and specifies a schema. Additional validations can be added. If two definitions for the same field
conflict (e.g. two different defaults are defined) Cue will show this as an error and it is invalid.

Also notice the top-level keys of `deployment` and `serviceaccount`. Because Cue is not Kubernetes
specific, it does not differentiate `kind` and `apiVersion` as special fields. So if we did not
separate the `Deployment` and `ServiceAccount` resources, Cue would attempt to merge the fields
and find conflicting definitions.

```
 ~/cue$ ls pkg/
 deployment.cue  serviceaccount.cue
 ~/cue$ cat pkg/deployment.cue
 package crm
 
 deployment: {
   apiVersion: "apps/v1",
   kind: "Deployment",
   metadata: name: "crm-deployment",
   spec: {
     selector: {
       matchLabels: {
         app: "crm"
       }
     }
     replicas: *1 | int,
     template: {
       metadata: {
         labels: {
           app: "crm",
         }
       }
       spec: {
         serviceAccount: "crm-serviceaccount",
         containers: [
           { name: "crm-api",
             image: "grasscrm:1.2",
             livenessProbe: {
               httpGet: {
                 path: "/",
                 port: "https",
               }
             }
             readinessProbe: {
               httpGet: {
                 path: "/",
                 port: "https",
               }
             }
           }]
        }
     }
   }
 }
 ~/cue$ cat pkg/serviceaccount.cue
 package crm
 
 serviceaccount: {
   apiVersion: "v1"
   kind: "ServiceAccount"
   metadata: name: "crm-serviceaccount"
 }
```

Then we can evaluate the documents. `--out yaml` specifies the resultant
documentation should be transpiled from Cue to YAML. Because Cue is not
YAML specific, it can support exporting results into various formats such
as JSON or YAML. The `-e serviceaccount -e deployment` unwinds the top-level
keys so that the final result is just the valid Kubernetes resource definitions.

```
 ~/cue$ cue eval --out yaml -e serviceaccount -e deployment pkg/*.cue
 apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: crm-serviceaccount
 ---
 apiVersion: apps/v1
 kind: Deployment
 metadata:
   name: crm-deployment
 spec:
   selector:
     matchLabels:
       app: crm
   replicas: 1
   template:
     metadata:
       labels:
         app: crm
     spec:
       serviceAccount: crm-serviceaccount
       containers:
       - name: crm-api
         image: grasscrm:1.2
         livenessProbe:
           httpGet:
             path: /
             port: https
         readinessProbe:
           httpGet:
             path: /
             port: http
```

### kpt

It was suggested I look at `kpt` as well, so I'm adding it into the comparison. 

Creating a package is pretty straightforward, similar to kustomize where the package
are just the plain resource definitions. It gets a little more complex to introduce
tunables as it uses OpenAPI-based configuration it terms `setters`.

Initially just create a basic package by using the `kpt` command to initialize
the package metadata in an empty directory and then pull in the YAML resource
definitions.

```
 ~/kpt/pkg$ kpt pkg init . --tag kpt.dev/app=grass --description "grass kpt package"
 writing Kptfile
 writing README.md
 ~/kpt/pkg$ ls
 Kptfile  README.md
 ~/kpt/pkg$ cp ../../target/pkg/crm-deployment.yaml .
 ~/kpt/pkg$ cp ../../target/pkg/crm-serviceaccount.yaml .
 ~/kpt/pkg$ ls
 crm-deployment.yaml  crm-serviceaccount.yaml  Kptfile  README.md
```

Now we need to make the Deployment replica count tunable by adding a 'setter'. The
`kpt` CLI can manipulate the files for you, or you can edit them by hand. The specification
appears in the `Kptfile`, then the references are added as line comments
in any number of YAML resource definitions.

```
 ~kpt/pkg$ cat Kptfile
 apiVersion: kpt.dev/v1alpha1
 kind: Kptfile
 metadata:
   name: .
 packageMetadata:
   tags:
   - kpt.dev/app=grass
   shortDescription: grass kpt package
 ~kpt/pkg$ cat crm-deployment.yaml
 ---
 apiVersion: apps/v1
 kind: Deployment
 metadata:
   name: crm-deployment
   labels:
     deployed-by: hand
 spec:
   selector:
     matchLabels:
       app: crm
   replicas: 1
   template:
     metadata:
       labels:
         app: crm
     spec:
       serviceAccount: crm-serviceaccount
       containers:
       - name: crm-api
         image: grasscrm:1.2
         livenessProbe:
           httpGet:
             path: /
             port: https
         readinessProbe:
           httpGet:
             path: /
             port: https
 ...
 ~kpt/pkg$ kpt cfg create-setter . replicas 1
 ~kpt/pkg$ cat Kptfile
 apiVersion: kpt.dev/v1alpha1
 kind: Kptfile
 metadata:
   name: .
 packageMetadata:
   tags:
   - kpt.dev/app=grass
   shortDescription: grass kpt package
 openAPI:
   definitions:
     io.k8s.cli.setters.replicas:
       x-k8s-cli:
         setter:
           name: replicas
           value: "1"
 ~kpt/pkg$ cat crm-deployment.yaml
 apiVersion: apps/v1
 kind: Deployment
 metadata:
   name: crm-deployment
   labels:
     deployed-by: hand
 spec:
   selector:
     matchLabels:
       app: crm
   replicas: 1 # {"$ref":"#/definitions/io.k8s.cli.setters.replicas"}
   template:
     metadata:
       labels:
         app: crm
     spec:
       serviceAccount: crm-serviceaccount
       containers:
       - name: crm-api
         image: grasscrm:1.2
         livenessProbe:
           httpGet:
             path: /
             port: https
         readinessProbe:
           httpGet:
             path: /
             port: https
```

We can add additional validation using OpenAPI schema fixtures. Similar to
the cue example, we can specify that `replicas` must be an int. Note the
additional `type` field under `io.k8s.cli.setters.replicas` below.

```
 ~/kpt/pkg$ cat Kptfile
 apiVersion: kpt.dev/v1alpha1
 kind: Kptfile
 metadata:
   name: .
 packageMetadata:
   tags:
   - kpt.dev/app=grass
   shortDescription: grass kpt package
 openAPI:
   definitions:
     io.k8s.cli.setters.replicas:
       type: integer
       x-k8s-cli:
         setter:
           name: replicas
           value: "1"
```

kpt offers powerful querying and reporting against a package of Kubernetes
resources, even if they aren't initialized into a kpt package. Here we can
see in a tree-view the image used for each deployment.

```
 ~/kpt/pkg$ kpt cfg tree . --image
 .
 ├── [crm-deployment.yaml]  Deployment crm-deployment
 │   └── spec.template.spec.containers
 │       └── 0
 │           └── image: grasscrm:1.2
 └── [crm-serviceaccount.yaml]  ServiceAccount crm-serviceaccoun
```

Here was dump the YAML resource definitions such that they could be
sent to `kubectl`, however `kpt` also supports directly applying
definitions to a cluster. Because the setter references are hidden
behind line comments, there is no issue including them in the serialization.
They will just be ignored by a YAML parser.

```
 ~kpt/pkg$ kpt cfg cat .
 apiVersion: apps/v1
 kind: Deployment
 metadata:
   name: crm-deployment
   labels:
     deployed-by: hand
 spec:
   replicas: 1 # {"$ref":"#/definitions/io.k8s.cli.setters.replicas"}
   selector:
     matchLabels:
       app: crm
   template:
     metadata:
       labels:
         app: crm
     spec:
       serviceAccount: crm-serviceaccount
       containers:
       - name: crm-api
         image: grasscrm:1.2
         livenessProbe:
           httpGet:
             port: https
             path: /
         readinessProbe:
           httpGet:
             port: https
             path: /
 ---
 apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: crm-serviceaccount
```

## Coming Next

In the next article we'll see how to build environment-specific
tuning in each tool. Also I'll talk about some ways CI can be
used for gating the configurations based on policies from one
or more sources.
