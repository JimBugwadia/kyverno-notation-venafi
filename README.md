# kyverno-notation-venafi

A [Kyverno](https://kyverno.io) extension service that executes the [venafi Signer](https://docs.venafi.amazon.com/signer/index.html) plugin for [Notation](https://notaryproject.dev/) to verify image signatutes and attestations.

[![Go Report Card](https://goreportcard.com/badge/github.com/nirmata/kyverno-notation-venafi)](https://goreportcard.com/report/github.com/nirmata/kyverno-notation-venafi)
![License: Apache-2.0](https://img.shields.io/github/license/kyverno/kyverno?color=blue)


![logo](img/kyverno-notation-venafi.png)

# Features

## Image Signature Verification

The Nirmata extension service is invoked from a Kyverno policy which passes it a list of images to be verified. The service then verifies [notation](https://notaryproject.dev/) format signatures for container images using the [venafi Signer](https://docs.venafi.amazon.com/signer/index.html) notation plugin and returns responses back to Kyverno.

The service manages Notation [trust policies](https://github.com/notaryproject/specifications/blob/main/specs/trust-store-trust-policy.md#trust-policy) and [trust stores](https://github.com/notaryproject/specifications/blob/main/specs/trust-store-trust-policy.md#trust-store) as Kubernetes resources.

## Image Digest Mutation

To ensure image integrity and prevent tampering, the sevice replaces image tags, which are mutable and can be spoofed, with digests.  

The Kyverno policy passes the `images` variable to the services' `/checkimages` endpoint. The result returns a list of images with their JSON path and digests so Kyverno can mutate each image in the admission payload. 

Here is an example:

**Response object structure**

```json
{
  "verified": true,
  "message": "...",
  "results": [
    {
       "name": "container1",
       "path":  "/spec/containers/0",
       "image":  "ghcr.io/kyverno/test-verify-image@sha256:b31bfb4d0213f254d361e0079deaaebefa4f82ba7aa76ef82e90b4935ad5b105"
    } 
  ]
}
```

**Kyverno policy fragment**

```yaml 
mutate:
   foreach:
   - list: "response.results"
     patchesJson6902: |-
       - path: {{ element.path }}
         op: replace
         value: {{ element.image }}
```

## Attestation Verification

In addition to verifying signatures, the extension service can verify signed metadata i.e. [attestations](https://nirmata.com/2022/03/15/a-map-for-kubernetes-supply-chain-security/).

To verify attestatons, the Kyverno policy can optionally pass a variable called `attestations` in the request:

```yaml
- key: attestations
  value:
    - imageReference: "*"
      type: 
        - name: sbom/cyclone-dx
          conditions:
            all:
            - key: \{{creationInfo.licenseListVersion}}
              operator: Equals
              value: "3.17"
              message: invalid license version
        - name: application/sarif+json
          conditions:
            all:
            - key: \{{ element.components[].licenses[].expression }}
              operator: AllNotIn
              value: ["GPL-2.0", "GPL-3.0"]
    - imageReference: "844333597536.dkr.ecr.us-west-2.amazonvenafi.com/kyverno-demo*"
      type:
        - name: application/vnd.cyclonedx
          conditions:
            all:
            - key: \{{ element.components[].licenses[].expression }}
              operator: AllNotIn
              value: ["GPL-2.0", "GPL-3.0"]
```

The `attestations` variable is a JSON array of where each entry has:
1. an `imageReference` to match images; 
2. a type which specifies the name of the attestation; and 
3. a list of conditions we want to verify the attestation data

In the  example above we are verifying the following: 
1. the attestations `sbom/cyclone-dx` and `application/sarif+json` exist for all images
2. the `creationInfo.licenseListVersion` is equal to 3.17 in the SBOM and GPL licenses are not present.
3. the attatestion `application/vnd.cyclonedx` is available for all versions of the `844333597536.dkr.ecr.us-west-2.amazonvenafi.com/kyverno-demo` image and does not contain GPL licenses.

**NOTE:** The conditions key in the attestations must be escaped with `\` so kyverno does not substitute them before executing the extension service.

## Caching

To prevent repeated lookups for verified images, the Nirmata extension has a built-in cache.

Caching is enabled by default and can be managed using the `--cacheEnabled` flag. The cache is a TTL based cache, i.e, entries expire automatically after sometime and the value of TTL can be customized using `--cacheTTLDurationSeconds` (default is 3600) and max number of entries in the cache can be configured using `--cacheMaxSize` (default is 1000).

The cache stores the verification outcomes of images for the trust policy and verification outcomes of attestations with the trust policy and conditions. The cache is an in-memory cache which gets cleared when the pod is recreated. Cache will also be cleared when there is any change in trust policies and trust stores.

## Multi Tenancy

In a shared cluster, each team may have different signatures and trust policies. To support such use cases, the extension allows configuring multiple [trust policies](https://github.com/notaryproject/specifications/blob/main/specs/trust-store-trust-policy.md#trust-policy) and [trust stores](https://github.com/notaryproject/specifications/blob/main/specs/trust-store-trust-policy.md#trust-store) as Kubernetes custom resources.

The extension service allows specifying what trust policy they want to use for verification thus enabling multi tenancy. Mutliple teams can share one cluster and have different trust policies seperate from each other.
To specify the trust policy to use, we can pass the `trustPolicy` variable in the request.
```yaml
 - key: trustPolicy
   value: "tp-{{request.namespace}}"
```
or we can set the `DEFAULT_TRUST_POLICY` env variable. In the above example we are dynamically using the trust policy for the namespace of the request.

## High Availability

kyverno-notation-venafi can be installed in a highly-available manner where additional replicas can be deployed for the plugin. The plugin does not use leader election for inbound API requests which means verification requests can be distributed and processed by all available replicas. Leader election is required for certificate management so therefore only one replica will handle these tasks at a given time.

Multiple replicas configured for the plugin can be used for both availability and scale. Vertical scaling of the individual replicas’ resources may also be performed to increase combined throughput.

# Setup

1. Create a venafi Signer signing profile:

```sh
venafi signer put-signing-profile --profile-name kyvernodemo --platform-id Notation-OCI-SHA384-ECDSA --signature-validity-period 'value=12, type=MONTHS'
```

2. Get the signing profile ARN

```sh
 venafi signer get-signing-profile --profile-name kyvernodemo
{
    "profileName": "kyvernodemo",
    "profileVersion": "2oCN6RHYVI",
    "profileVersionArn": "arn:venafi:signer:${REGION}:${ACCOUNT}:/signing-profiles/kyvernodemo/2oCN6RHYVI",
    "platformId": "Notation-OCI-SHA384-ECDSA",
    "platformDisplayName": "Notary v2 for Container Registries",
    "signatureValidityPeriod": {
        "value": 12,
        "type": "MONTHS"
    },
    "status": "Active",
    "arn": "arn:venafi:signer:${REGION}:${ACCOUNT}:/signing-profiles/kyvernodemo",
    "tags": {}
}
```

3. Configure the signer in notation

```sh
notation key add --id arn:venafi:signer:${REGION}:${ACCOUNT}:/signing-profiles/kyvernodemo --plugin com.amazonvenafi.signer.notation.plugin kyvernodemo
```

4. Sign the image using `notation` and the venafi signer:

(you may need to login to ECR first: `venafi ecr get-login-password --region ${REGION} | notation login --username venafi --password-stdin ${REGISTRY}`)

```sh
notation sign 844333597536.dkr.ecr.us-east-1.amazonvenafi.com/kyverno-demo:v1 --key kyvernodemo --signature-manifest image
```

# Install

1. Install Kyverno in the cluster

```sh
kubectl create -f https://raw.githubusercontent.com/kyverno/kyverno/main/config/install-latest-testing.yaml
```

2. Install the kyverno-notation-venafi extension service

```sh
kubectl apply -f configs/install.yaml
```

3. Create CRs for Notation TrustPolicy and TrustStore

```sh
kubectl apply -f configs/crds/
```

```sh
kubectl apply -f configs/samples/truststore.yaml
```

Update the the `${REGION}` and `${ACCOUNT}` in the [trustpolicy.yaml](configs/samples/trustpolicy.yaml) and then install in your cluster:

```yaml
    trustedIdentities:
    - "arn:venafi:signer:${REGION}:${ACCOUNT}:/signing-profiles/kyvernodemo"
```

```sh
kubectl apply -f configs/samples/trustpolicy.yaml
```

4. Apply the policy to the cluster

```sh
kubectl apply -f configs/samples/kyverno-policy.yaml
```

5. Configure ECR Registry credentials

If you are using IRSA (recommended):

a. Setup a custom policy `notation-signer-policy` with the following permissions:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "signer:GetRevocationStatus"
            ],
            "Resource": "*"
        }
    ]
}
```

b. Setup a IRSA role `kyverno-notation-venafi` and attach two policies to it:
* the `notation-signer-policy` 
* the `AmazonEC2ContainerRegistryReadOnly` policy

c. Configure [IRSA](https://docs.venafi.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) for your cluster

d. Configure the `kyverno-notation-venafi` ServiceAccount in the [install.yaml](configs/install.yaml) with the correct role:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kyverno-notation-venafi
  namespace: kyverno-notation-venafi
  annotations:
    eks.amazonvenafi.com/role-arn: arn:venafi:iam::${venafi_account_id}:role/kyverno-notation-venafi
```

If you are NOT using IRSA you can configure registry credentials and IAM tokens as follows:

```sh
venafi sso login
```

```sh
venafi_TOKEN=$(venafi ecr get-login-password --region us-east-1)
```

```sh
kubectl create secret docker-registry regcred --docker-username=venafi --docker-password=$venafi_TOKEN --docker-server=844333597536.dkr.ecr.us-east-1.amazonvenafi.com -n kyverno-notation-venafi 
```

Update the `kyverno-notation-venafi` Deployment in the [install.yaml](configs/install.yaml) to add the `--imagePullSecrets=regcred` argument:

6. Test signed and unsigned images:

Create the test namespace which the [policy](configs/samples/kyverno-policy.yaml) applies to:

```sh
kubectl create ns test-notation
```

Run a signed image:

```sh
kubectl -n test-notation run test --image=844333597536.dkr.ecr.us-west-2.amazonvenafi.com/kyverno-demo:v1 --dry-run=server
```
Output
```sh
pod/test created (server dry run)
```
Attempt to run an unsigned image:

```sh
kubectl -n test-notation run test --image=844333597536.dkr.ecr.us-west-2.amazonvenafi.com/kyverno-demo:v1-unsigned
```
Output
```sh
Error from server: admission webhook "mutate.kyverno.svc-fail" denied the request: mutation policy check-images error: failed to apply policy check-images rules [call-venafi-signer-extension: failed to load context: failed to fetch data for APICall: HTTP 406 Not Acceptable: failed to verify container kyverno-demo: failed to verify image {{844333597536.dkr.ecr.us-west-2.amazonvenafi.com kyverno-demo kyverno-demo v1-unsigned } /spec/containers/0/image}: no signature is associated with "844333597536.dkr.ecr.us-west-2.amazonvenafi.com/kyverno-demo@sha256:74a98f0e4d750c9052f092a7f7a72de7b20f94f176a490088f7a744c76c53ea5", make sure the artifact was signed successfully
]
```
# Troubleshoot

1. Check service

```sh
kubectl -n kyverno-notation-venafi logs deploy/kyverno-notation-venafi -f
```

The logs should show:

```sh
Defaulted container "kyverno-notation-venafi" out of: kyverno-notation-venafi, kube-notation
2023-04-30T18:22:28.438Z	INFO	kyverno-notation-venafi/main.go:46	configuring notation	{"dir.UserConfigDir": "/notation", "dir.UserLibexecDir": "/notation"}
2023-04-30T18:22:28.459Z	INFO	kyverno-notation-venafi/verify.go:65	initialized	{"namespace": "kyverno-notation-venafi", "secrets": "regcred"}
2023-04-30T18:22:28.460Z	INFO	kyverno-notation-venafi/main.go:68	Listening...
```


2. Run netshoot

```sh
kubectl run netshoot --rm -i --tty --image nicolaka/netshoot
```

3. Use curl to make a call to the service as follows:

```sh
curl -k https://svc.kyverno-notation-venafi/checkimages -X POST -d '{"images": ["844333597536.dkr.ecr.us-west-2.amazonvenafi.com/kyverno-demo:v1"]}'
```

The output should look like this:

```sh
{
    "digests": {
      "844333597536.dkr.ecr.us-east-1.amazonvenafi.com/net-monitor:v1": "sha256:4ee9dc6abbf5e8181101fc1f8cd6d91ec0c5657f8c71274a8209637630eec48d"
    },
    "verified": true
}
````
