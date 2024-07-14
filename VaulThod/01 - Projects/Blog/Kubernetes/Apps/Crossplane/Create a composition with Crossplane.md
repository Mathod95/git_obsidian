---
createdAt: 2024-05-15T03:28:00
status: TODO
tags:
  - APP/CROSSPLANE
---
# Introduction
# Objectifs
# Prérequis
# Ma configuration
---
# How to create a composition with crossplane

Crossplane has four core components that users commonly mix up:

- Compositions: A template to define how to create resources.

- [Composite Resource Definition (XRD)](https://docs.crossplane.io/latest/concepts/composite-resource-definitions/): A custom API specification. __We will call that a definition__

- [Composite Resource (XR)](https://docs.crossplane.io/latest/concepts/composite-resources/): Created by using the custom API defined in a Composite Resource Definition. XRs use the Composition template to create new managed resources.

- [Claims (XRC)](https://docs.crossplane.io/latest/concepts/claims/) - Like a Composite Resource, but with namespace scoping.

Here's how to create these compositions in the crossplane repo

    .

    └── compositions

        ├── eks

        │   ├── claim.yaml

        │   ├── composition.yaml

        │   └── definitions.yaml

        ├── s3

        │   ├── claim.yaml

        │   ├── composition.yaml

        │   └── definitions.yaml

        └── rds

            ├── claim.yaml

            ├── composition.yaml

            └── definitions.yaml

If you’re coming from the Terraform world (I'm talking to you @KennySalibovitchky) you can think of an XRD as similar to the variable blocks of a Terraform module, while the Composition is the rest of the module’s HCL code that describes how to use those variables to create a bunch of resources. In this analogy the XR or claim is a little like a tfvars file providing inputs to the module.

# Provision a S3 Bucket with Crossplane

__The crossplane core Controller and the Provider AWS Controller should now be ready to provision any infrastructure component in AWS!__

Therefore we can have a look into the Crossplane AWS provider API docs:https://doc.crds.dev/github.com/upbound/provider-aws/s3.aws.upbound.io/Bucket/v1beta1@v0.45.0

## Defining a CompositeResourceDefinition (XRD) for our S3 Bucket

> A CompositeResourceDefinition (or XRD) defines the type and schema of your XR. It lets Crossplane know that you want a particular kind of XR to exist, and what fields that XR should have.

Since defining your own CompositeResourceDefinitions and Compositions is the main work todo with Crossplane, it's always good to know the full Reference documentation which can be found here https://docs.crossplane.io/v1.14/concepts/compositions/

One of the things to know is that Crossplane automatically injects some common 'machinery' into the manifests of the XRDs and Compositions: https://docs.crossplane.io/v1.14/concepts/composite-resources/

All possible fields an XRD can have are documented here:

https://docs.crossplane.io/v1.14/concepts/composite-resource-definitions/

The field `spec.versions.schema` must contain a OpenAPI schema, which is similar to the ones used by any Kubernetes CRDs. They determine what fields the XR (and claim) will have. The full CRD documentation and a guide on how to write OpenAPI schemas could be found in the Kubernetes docs: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

Note that Crossplane will be automatically extended this section. Therefore the following fields are used by Crossplane and will be ignored if they're found in the schema:

    spec.resourceRef
    spec.resourceRefs
    spec.claimRef
    spec.writeConnectionSecretToRef
    status.conditions
    status.connectionDetails

So our Composite Resource Definition (XRD) for our S3 Bucket could look like [compositions/s3/definitions.yaml](compositions/s3/definitions.yaml):

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  # XRDs must be named 'x<plural>.<group>'
  name: xobjectstorages.crossplane.evina
spec:
  # This XRD defines an XR in the 'crossplane.evina' API group.
  # The XR or Claim must use this group together with the spec.versions[0].name as it's apiVersion, like this:
  # 'crossplane.evina/v1alpha1'
  group: crossplane.evina
  # XR names should always be prefixed with an 'X'
  names:
    kind: XObjectStorage
    plural: xobjectstorages
  # This type of XR offers a claim, which should have the same name without the 'X' prefix
  claimNames:
    kind: ObjectStorage
    plural: objectstorages
  # default Composition when none is specified (must match metadata.name of a provided Composition)
  # e.g. in composition.yaml
  defaultCompositionRef:
    name: objectstorage-composition
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    # OpenAPI schema (like the one used by Kubernetes CRDs). Determines what fields
    # the XR (and claim) will have. Will be automatically extended by crossplane.
    # See https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
    # for full CRD documentation and guide on how to write OpenAPI schemas
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            # We define 2 needed parameters here one has to provide as XR or Claim spec.parameters
            properties:
              parameters:
                type: object
                properties:
                  bucketName:
                    type: string
                  region:
                    type: string
                required:
                  - bucketName
                  - region
```

Install the XRD into our cluster with:

```shell ln:false hl:1
kubectl apply -f compositions/s3/definitions.yaml
```

We can double check the CRDs beeing created with `kubectl get crds` and filter them using `grep` to our group name `crossplane.evina`:

```shell ln:false hl:1
kubectl get crds | grep crossplane.evina
objectstorages.crossplane.evina                                2024-01-04T15:38:50Z
xobjectstorages.crossplane.evina                               2024-01-04T15:38:50Z
```

---

## Craft a Composition to manage our needed cloud resources

The main work in Crossplane has to be done crafting the Compositions. This is because they interact with the infrastructure primitives the cloud provider APIs provide.

Detailled docs to many of the possible manifest configurations can be found here https://docs.crossplane.io/latest/concepts/compositions/

https://marketplace.upbound.io/providers/upbound/provider-aws-s3/latest

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: objectstorage-composition
  labels:
    # An optional convention is to include a label of the XRD. This allows
    # easy discovery of compatible Compositions.
    crossplane.io/xrd: xobjectstorages.crossplane.jonashackt.io
    # The following label marks this Composition for AWS. This label can
    # be used in 'compositionSelector' in an XR or Claim.
    provider: aws
spec:
  # Each Composition must declare that it is compatible with a particular type
  # of Composite Resource using its 'compositeTypeRef' field. The referenced
  # version must be marked 'referenceable' in the XRD that defines the XR.
  compositeTypeRef:
    apiVersion: crossplane.jonashackt.io/v1alpha1
    kind: XObjectStorage
  # When an XR is created in response to a claim Crossplane needs to know where
  # it should create the XR's connection secret. This is configured using the
  # 'writeConnectionSecretsToNamespace' field.
  writeConnectionSecretsToNamespace: crossplane-system
  # Each Composition must specify at least one composed resource template.
  resources:
    # Providing a unique name for each entry is good practice.
    # Only identifies the resources entry within the Composition. Required in future crossplane API versions.
    - name: bucket
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/Bucket/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        metadata: {}
        spec:
          deletionPolicy: Delete
      patches:
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"
    - name: bucketpublicaccessblock
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/BucketPublicAccessBlock/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketPublicAccessBlock
        spec:
          forProvider:
            blockPublicAcls: false
            blockPublicPolicy: false
            ignorePublicAcls: false
            restrictPublicBuckets: false
      patches:
        - fromFieldPath: "spec.parameters.bucketPABName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "spec.forProvider.bucketRef.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"
    - name: bucketownershipcontrols
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/BucketOwnershipControls/v1beta1#doc:spec-forProvider-rule-objectOwnership
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketOwnershipControls
        spec:
          forProvider:
            rule:
              - objectOwnership: ObjectWriter
      patches:
        - fromFieldPath: "spec.parameters.bucketOSCName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "spec.forProvider.bucketRef.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"
    - name: bucketacl
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/BucketACL/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketACL
        spec:
          forProvider:
            acl: "public-read"
      patches:
        - fromFieldPath: "spec.parameters.bucketAclName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "spec.forProvider.bucketRef.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"
    - name: bucketwebsiteconfiguration
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/BucketWebsiteConfiguration/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketWebsiteConfiguration
        spec:
          forProvider:
            indexDocument:
              - suffix: index.html
      patches:
        - fromFieldPath: "spec.parameters.bucketWebConfName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "spec.forProvider.bucketRef.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"
  # If you find yourself repeating patches a lot you can group them as a named
  # 'patch set' then use a PatchSet type patch to reference them.
  #patchSets:
```

Let's create Composition via:

```shell hl:1 ln:false
kubectl apply -f compositions/s3/composition.yaml
```

---

## Craft a Composite Resource (XR) or Claim (XRC)

Crossplane could look quite intimidating when having a first look. There are few guides around to show how to approach a setup when using Crossplane the first time. You can choose between writing an XR __OR__ XRC! You don't need both, since the XR will be generated from the XRC, if you choose to craft a XRC.

https://docs.crossplane.io/v1.14/concepts/composite-resources/

Since we want to create a S3 Bucket, here's an suggestion for an [claim.yaml](crossplane-contrib/provider-aws/s3/claim.yaml):

```yaml
# Use the spec.group/spec.versions[0].name defined in the XRD
apiVersion: crossplane.jonashackt.io/v1alpha1
kind: ObjectStorage
metadata:
  # Only claims are namespaced, unlike XRs.
  namespace: default
  name: managed-s3
spec:
  # The compositionRef specifies which Composition this XR will use to compose
  # resources when it is created, updated, or deleted.
  compositionRef:
    name: objectstorage-composition
  # Parameters for the Composition to provide the Managed Resources (MR) with
  # to create the actual infrastructure components
  parameters:
    bucketName: microservice-ui-nuxt-js-static-bucket2
    region: eu-central-1
```

Testdrive with:

```shell ln:false hl:1
kubectl apply -f crossplane-contrib/provider-aws/s3/claim.yaml
```

When somthing goes wrong with the validation, this could look like this:

```shell ln:false hl:1 error:3
kubectl apply -f claim.yaml

error: error validating "claim.yaml": error validating data: [ValidationError(S3Bucket.metadata): unknown field "crossplane.io/external-name" in io.k8s.apimachinery.pkg.apis.meta.v1.ObjectMeta_v2, ValidationError(S3Bucket.spec): unknown field "parameters" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec, ValidationError(S3Bucket.spec.writeConnectionSecretToRef): missing required field "namespace" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec.writeConnectionSecretToRef, ValidationError(S3Bucket.spec): missing required field "bucketName" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec, ValidationError(S3Bucket.spec): missing required field "region" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec]; if you choose to ignore these errors, turn validation off with --validate=false
```

The Crossplane validation is a great way to debug your yaml configuration - it hints you to the actual problems that are still present.

### Waiting for resources to become ready

There are some possible things to check while your resources (may) get deployed after running a `kubectl apply -f claim.yaml` (see https://crossplane.io/docs/v1.8/getting-started/provision-infrastructure.html#claim-your-infrastructure).

The best overview gives a `kubectl get crossplane` which will simply list all the crossplane resources:

```shell
kubectl get crossplane

Warning: Please use v1beta1 version of this resource that has identical schema.

NAME                                                                                          ESTABLISHED   OFFERED   AGE

compositeresourcedefinition.apiextensions.crossplane.io/xs3buckets.crossplane.jonashackt.io   True          True      23m

  

NAME                                               AGE

composition.apiextensions.crossplane.io/s3bucket   2d17h

  

NAME                                      INSTALLED   HEALTHY   PACKAGE                           AGE

provider.pkg.crossplane.io/provider-aws   True        True      crossplanecontrib/provider-aws:v0.34.0   4d21h

  

NAME                                                           HEALTHY   REVISION   IMAGE                             STATE    DEP-FOUND   DEP-INSTALLED   AGE

providerrevision.pkg.crossplane.io/provider-aws-2189bc61e0bd   True      1          crossplanecontrib/provider-aws:v0.34.0   Active                               4d21h

  

NAME                                        AGE     TYPE         DEFAULT-SCOPE

storeconfig.secrets.crossplane.io/default   5d23h   Kubernetes   crossplane-system

```

* `kubectl get claim`: get all resources of all claim kinds, like PostgreSQLInstance.
* `kubectl get composite`: get all resources that are of composite kind, like XPostgreSQLInstance.
* `kubectl get managed`: get all resources that represent a unit of external infrastructure.

* `kubectl get <name-of-provider>`: get all resources related to <provider>.

* `kubectl get crossplane`: get all resources related to Crossplane.

We can also check our claim with `kubectl get <claim-kind> <claim-metadata.name>` like this:

```shell

$ kubectl get ObjectStorage managed-s3

NAME         READY   CONNECTION-SECRET               AGE

managed-s3           managed-s3-connection-details   5s

```
```shell 
zae
```
To watch the provisioned resources become ready we can run `kubectl get crossplane -l crossplane.io/claim-name=<claim-metadata.name>`:

```shell

kubectl get crossplane -l crossplane.io/claim-name=managed-s3

```

Check if the S3 Bucket has been created successfully via aws CLI with `aws s3 ls | grep microservice-ui-nuxt-js-static-bucket2`.



` ` `shell

$ aws s3 ls | grep microservice-ui-nuxt-js-static-bucket2

2022-06-27 11:56:26 microservice-ui-nuxt-js-static-bucket2

```

## Troubleshooting your crossplane configuration

https://docs.crossplane.io/knowledge-base/guides/troubleshoot/

> Per Kubernetes convention, Crossplane keeps errors close to the place they happen. This means that if your claim is not becoming ready due to an issue with your Composition or with a composed resource you’ll need to “follow the references” to find out why. Your claim will only tell you that the XR is not yet ready.

The docs also tell us what they mean by "follow the references":

* Find your XR by running `kubectl describe <claim-kind> <claim-metadata.name>` and look for its “Resource Ref” (aka `spec.resourceRef`).

* Run `kubectl describe` on your XR. This is where you’ll find out about issues with the Composition you’re using, if any.

* If there are no issues but your XR doesn’t seem to be becoming ready, take a look for the “Resource Refs” (or `spec.resourceRefs`) to find your composed resources.

* Run `kubectl describe` on each referenced composed resource to determine whether it is ready and what issues, if any, it is encountering.

## Remove your App

Finally remove our S3 Bucket with:

```shell 
kubectl delete -f upbound/provider-aws-s3/claim.yaml
```

## Publish Crossplane Configuration Package into GitHub Container Registry

Let's package our Composite Resource definitions as a Configuration. Therefore we need to use the Crossplane CLI via <code>kubectl crossplane build configuration</code> (now they are a Package) - and push them to an OCI registry via <code>kubectl crossplane push configuration</code>. With this Configurations can also be easily installed into other Crossplane clusters.

Therefore we need a `crossplane.yaml` file as described in https://crossplane.io/docs/v1.8/getting-started/create-configuration.html#build-and-push-the-configuration

See also https://crossplane.io/docs/v1.8/concepts/packages.html#configuration-packages

Our [crossplane-contrib/provider-aws/s3/crossplane.yaml](crossplane-contrib/provider-aws/s3/crossplane.yaml) is of `kind: Configuration` and defines the minimum crossplane version needed alongside the crossplane AWS provider:

```yaml

apiVersion: meta.pkg.crossplane.io/v1

kind: Configuration

metadata:

  name: s3-bucket-composition

spec:

  crossplane:

    version: ">=v1.8"

  dependsOn:

    - provider: crossplanecontrib/provider-aws

      version: ">=v0.33.0"

```

Having this `crossplane.yaml` in place we can build our Configuration at last. On a command line go into the directory where the `crossplane.yaml` resides and run the `kubectl crossplane build` command:

```shell

$ cd crossplane-s3

$ kubectl crossplane build configuration

```

Really strange, getting

```shell

kubectl crossplane build configuration

kubectl crossplane: error: failed to build package: failed to parse package: {path:/Users/jonashecht/dev/kubernetes/crossplane-awws-azure/crossplane-contrib/provider-aws/s3/composition.yaml position:0}: no kind "S3Bucket" is registered for version "crossplane.jonashackt.io/v1alpha1" in scheme "/home/runner/work/crossplane/crossplane/internal/xpkg/scheme.go:47"

```

---

## Use Upbound Platform Reference Architecture AWS to provision a EKS cluster

__What I didn't wanted to do is to duplicate some code__ into my `Composition` from all those sources I found (mainly this https://www.kloia.com/blog/production-ready-eks-cluster-with-crossplane, this https://aws.amazon.com/blogs/containers/gitops-model-for-provisioning-and-bootstrapping-amazon-eks-clusters-using-crossplane-and-argo-cd/ and https://aws.amazon.com/blogs/opensource/introducing-aws-blueprints-for-crossplane/) about provisioning a EKS cluster with crossplane.

### Install platform-ref-aws

https://github.com/upbound/platform-ref-aws#install-the-platform-configuration

```shell

S3 creation error: CannotCreateExternalResource managed/bucket.s3.aws.crossplane.io failed to create the Bucket: InvalidBucketAclWithObjectOwnership: Bucket cannot have ACLs set with ObjectOwnership's BucketOwnerEnforced setting

```

we need to switch over to the official AWS provider based on Upjet. Do we need to use the UXP (Universal Control Plane) flavour of Crossplane for that? No, the docs https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/docs/configuration state:

> The Upbound AWS official provider may also be used with upstream Crossplane.

So let's use the provider inside our project!