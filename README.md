# api
The canonical location of the OpenShift API definition.
This repo holds the API type definitions and serialization code used by [openshift/client-go](https://github.com/openshift/client-go)
APIs in this repo ship inside OCP payloads.

## defining new APIs

When defining a new API, please follow [the OpenShift API
conventions](https://github.com/openshift/enhancements/blob/master/CONVENTIONS.md#api),
and then follow the instructions below to regenerate CRDs (if necessary) and
submit a pull request with your new API definitions and generated files.

### required labels

In addition to the standard `lgtm` and `approved` labels this repository requires either:

`bugzilla/valid-bug` - applied if your PR references a valid bugzilla bug

OR

`qe-approved`, `docs-approved`, and `px-approved` - these labels can be applied by anyone in the openshift org via the `/label` command.

Who should apply these qe/docs/px labels?
- For a no-FF team who is merging a feature before code freeze, they need to get those labels applied to their api repo PR by the appropriate teams (i.e. qe, docs, px)
- For a FF(traditional) team who is merging a feature before FF, they can self-apply the labels(via /label commands), they are basically irrelevant for those teams
- For a FF team who is merging a feature after FF, the PR should be rejected barring an exception

Why are these labels needed?

We need a way for no-FF teams to be able to merge post-FF that does not require a BZ.  For non-shared repos that mechanism is the 
qe/docs/px-approved labels.  We are expanding that mechanism to shared repos because the alternative would be that no-FF teams would
put a dummy `bugzilla/valid-bug` label on their feature PRs in order to be able to merge them after feature freeze.  Since most
individuals can't apply a `bugzilla/valid-bug` label to a PR, this introduces additional obstacles on those PRs.  Conversely, anyone
can apply the docs/qe/px-approved labels, so "FF" teams that need to apply these labels to merge can do so w/o needing to involve
anyone additional.

Does this mean feature-freeze teams can use the no-FF process to merge code?

No, signing a team up to be a no-FF team includes some basic education on the process and includes ensuring the associated QE+Docs
participants are aware the team is moving to that model.  If you'd like to sign your team up, please speak with Gina Hargan who will
be happy to help on-board your team.

## generating CRD schemas

Since Kubernetes 1.16, every CRD created in `apiextensions.k8s.io/v1` is required to have a [structural OpenAPIV3 schema](https://kubernetes.io/blog/2019/06/20/crd-structural-schema/). The schemas provide server-side validation for fields, as well as providing the descriptions for `oc explain`. Moreover, schemas ensure structural consistency of data in etcd. Without it anything can be stored in a resource which can have security implications. As we host many of our CRDs in this repo along with their corresponding Go types we also require them to have schemas. However, the following instructions apply for CRDs that are not hosted here as well.

These schemas are often very long and complex, and should not be written by hand. For OpenShift, we provide Makefile targets in [build-machinery-go](https://github.com/openshift/build-machinery-go/) which generate the schema, built on upstream's [controller-gen](https://github.com/kubernetes-sigs/controller-tools) tool.

If you make a change to a CRD type in this repo, simply calling `make update-codegen-crds` should regenerate all CRDs and update the manifests. If yours is not updated, ensure that the path to its API is included in our [calls to the Makefile targets](https://github.com/openshift/api/blob/release-4.5/Makefile#L17-L29), if this doesn't help try calling `make generate-with-container` for executing the generators in a controlled environment.

To add this generator to another repo:
1. Vendor `github.com/openshift/build-machinery-go`

2. Update your `Makefile` to include the following:
```
include $(addprefix ./vendor/github.com/openshift/build-machinery-go/make/, \
  targets/openshift/crd-schema-gen.mk \
)

$(call add-crd-gen,<TARGET_NAME>,<API_DIRECTORY>,<CRD_MANIFESTS>,<MANIFEST_OUTPUT>)
```
The parameters for the call are:

1. `TARGET_NAME`: The name of your generated Make target. This can be anything, as long as it does not conflict with another make target. Recommended to be your api name.
2. `API_DIRECTORY`: The location of your API. For example if your Go types are located under `pkg/apis/myoperator/v1/types.go`, this should be `./pkg/apis/myoperator/v1`.
3. `CRD_MANIFESTS`: The directory your CRDs are located in. For example, if that is `manifests/my_operator.crd.yaml` then it should be `./manifests`
4. `MANIFEST_OUTPUT`: This should most likely be the same as `CRD_MANIFESTS`, and is only provided for flexibility to output generated code to a different directory.

You can include as many calls to different APIs as necessary, or if you have multiple APIs under the same directory (eg, `v1` and `v2beta1`) you can use 1 call to the parent directory pointing to your API.

After this, calling `make update-codegen-crds` should generate a new structural OpenAPIV3 schema for your CRDs.

**Notes** 
- This will not generate entire CRDs, only their OpenAPIV3 schemas. If you do not already have a CRD, you will get no output from the generator.
- Ensure that your API is correctly declared for the generator to pick it up. That means, in your `doc.go`, include the following:
  1. `// +groupName=<API_GROUP_NAME>`, this should match the `group` in your CRD `spec`
  2. `// +kubebuilder:validation:Optional`, this tells the operator that fields should be optional unless explicitly marked with `// +kubebuilder:validation:Required`
  
For more information on the API markers to add to your Go types, see the [Kubebuilder book](https://book.kubebuilder.io/reference/markers.html)

### Order of generation
`make update-codegen-crds` does roughly this:

1. Run the `empty-partial-schema` tool.  This creates empty CRD manifests in `zz_generated.featuregated-crd-manifests` for each FeatureGate.
2. Run the `schemapatch` tool.  This fills in the schema for each per-FeatureGate CRD manifest.
3. Run the `manifest-merge` tool.  This combines all the per-FeatureGate CRD manifests and `manual-overrides`

#### empty-partial-schema
This tool is gengo based and scans all types for a `// +kubebuilder:object:root=true` marker.
For each type match, the type is navigated and all tags that include a `featureGate`
(`// +openshift:enable:FeatureGate`, `// +openshift:validation:FeatureGateAwareEnum`, and `// +openshift:validation:FeatureGateAwareXValidation`)
are tracked.
For each type, for each FeatureGate, a file CRD manifest is created in `zz_generated.featuregated-crd-manifests`.
The most common kube-builder tags are re-implemented in this stage to fill in the non-schema portion of the CRD manifests.
This includes things like metadata, resource, and some custom openshift tags as well.

The generator ignores the schema when doing verify, so it doesn't fail on needing to run `schemapatch`.
The generator should clean up old FeatureGated manifests when the gate is removed.
Ungated files are created for resources that are sometimes ungated.
Annotations are injected to indicate which FeatureGate a manifest is for: this is later read by `schemapatch` and `manifest-merge`.

#### schemapatch
This tool is kubebuilder based with patches to handle FeatureGated types, members, and validation.
It reads the injected annotation from `empty-partial-schema` to decide which FeatureGate should be considered enabled when
creating the schema that needs to be injected.
It has no knowledge of whether the FeatureGate is enabled or disabled in particular ClusterProfile,FeatureSet tuples.
It only needs a single pass over all the FeatureGated partial manifests.

If the schema generation isn't doing what you want, `manual-override-crd-manifests` allows partially overlaying bits of the CRD manifest.
`yamlpatch` is no longer supported.
The format is just "write the CRD you want and delete the stuff the generator sets properly".
More specifically, it is the partial manifest that server-side-apply (structured merge diff) would properly merge on top of
the CRD that is generated otherwise.
Caveat, you cannot test this with a kube-apiserver because the CRD schema uses atomic lists and we had to patch that
schema to indicate map lists keyed by version.

#### manifest-merge
This tool is gengo based and it combines the files in `zz_generated.featuregated-crd-manifests` and `manual-override-crd-manifests`
on a per ClusterProfile,FeatureSet tuple.
This tool takes as input all possible ClusterProfiles and all possible FeatureSets.
It then maps from ClusterProfile,FeatureSet tuple to the set of enabled and disabled FeatureGates.
Then for each CRD,ClusterProfile,Feature tuple, it merges the pertinent input using structured-merge-diff (SSA) logic
based on the CRD schema plus a patch to make atomic fields map-lists.
Pertinence is determined based on
1. does this manifest have preferred ClusterProfile annotations: if so, honor them; if not, include everywhere.
2. does this manifest have FeatureGate annotations: if so, match against the enabled set for the ClusterProfile,FeatureSet tuple.
   Note that CustomNoUpgrade selects everything

Once we have CRD for each ClusterProfile,FeatureSet tuple we choose what to serialize.
This roughly follows:
1. if all the CRDs are the same, write a single file and annotate with no FeatureSet and every ClusterProfile. Done.
2. if all the CRDs are the same across all ClusterProfiles for each FeatureSet, create one file per FeatureSet and
   annotate with one FeatureSet and all ClusterProfiles. Done.
3. if all the CRDs are the same across all FeatureSets for one ClusterProfile, create one file and annotate
   with no FeatureSet and one ClusterProfile. Continue to 4.
4. for all remaining ClusterProfile,FeatureSet tuples, serialize a file with one FeatureSet and one ClusterProfile.

