Open Policy Container demo steps

login to the container registry

```
gh auth token | policy login -s ghcr.io -u $USER --password-stdin
```

```
Logging in.
server: ghcr.io
user: gertd

OK.
```

lets create a policy to play with

list available templates

```
policy templates list
```

```
Fetching templates 

  NAME             KIND    DESCRIPTION                    
  github           cicd    GitHub policy CI/CD template.  
  gitlab           cicd    GitLab policy CI/CD template.  
  policy-template  policy  Minimal policy template. 
```

lets create a policy

```
policy templates apply policy-template
```

```
Processing template 'policy-template' 

Generating files .

The template 'policy-template' was created successfully.
```

what did it create?

```
tree

 .
├──  README.md
└──  src
    └──  policies
        └──  hello-rego.rego
```

lets see if we can build the policy and create a policy image

```
policy build . -t demo-policy:0.0.0
```

```
Created new image.
digest: sha256:0f66afb3eaafbb9056a1bc1e5bc90ecdf6bcbdf9288f6e2e398cd3c23ae71a83

Tagging image.
reference: ghcr.io/demo-policy:0.0.0
```

where did it go? 
```
policy images
```

```
  REPOSITORY   TAG    IMAGE ID      CREATED               SIZE  
  demo-policy  0.0.0  e583dc0e018f  2024-02-02T04:13:29Z  736B  
```

lets load the policy from the local store and run it using policy repl

```
policy repl ghcr.io/demo-policy:0.0.0
```

```
running policy [ghcr.io/demo-policy:0.0.0]
> data
{
  "policies": {
    "hello": {
      "allowed": false,
      "enabled": false,
      "visible": false
    }
  }
}
> exit
``` 

lets add some CI, so we can build and publish our policies

```
policy templates apply github
```

```
Processing template 'github' 

> server (ghcr.io): 

> user (): gertd

> secret name (TOKEN): GHCR_PUSH_TOKEN

> org/repo: gertd/demo

Generating files .

The template 'github' was created successfully.
```

lets use the policy in OPA

```
opa run -s -c config-opa.yaml
```

config-opa.yaml
```
services:
  ghcr-registry:
    url: https://ghcr.io
    type: oci

bundles:
  authz:
    service: ghcr-registry
    resource: ghcr.io/gertd/demo:0.0.0
    persist: true
    polling:
      min_delay_seconds: 60
      max_delay_seconds: 120

persistence_directory: ./cache
```

lets use the policy in topaz

```
topaz configure -n demo -r ghcr.io/gertd/demo:0.0.0 --force
```

```
>>> configure policy

certs directory: /Users/gertd/.config/topaz/certs

  FILE            ACTION                        
  gateway.crt     skipped, file already exists  
  gateway-ca.crt  skipped, file already exists  
  gateway.key     skipped, file already exists  
  grpc.crt        skipped, file already exists  
  grpc-ca.crt     skipped, file already exists  
  grpc.key        skipped, file already exists  
policy name: demo
```

start the engine...

```
topaz start
```

```
>>> starting topaz...
a1f96767dbba2f02384f8951ca67ba024298068c448e71e8fdec5e179d27f4ab
```

and explore the policy inside the console

```
topaz console
```

lets sign the policy

first we have to login to docker, to establish a security context cosign can use

```
gh auth token | docker login -u $USER ghcr.io --password-stdin
```

initialize

```
cosign initialize
```

```
Root status: 
 {
        "local": "/Users/gertd/.sigstore/root",
        "remote": "https://tuf-repo-cdn.sigstore.dev",
        "metadata": {
                "root.json": {
                        "version": 8,
                        "len": 5406,
                        "expiration": "26 Mar 24 04:38 UTC",
                        "error": ""
                },
                "snapshot.json": {
                        "version": 124,
                        "len": 2300,
                        "expiration": "20 Feb 24 16:07 UTC",
                        "error": ""
                },
                "targets.json": {
                        "version": 8,
                        "len": 5254,
                        "expiration": "26 Mar 24 05:51 UTC",
                        "error": ""
                },
                "timestamp.json": {
                        "version": 154,
                        "len": 721,
                        "expiration": "06 Feb 24 16:07 UTC",
                        "error": ""
                }
        },
        "targets": [
                "artifact.pub",
                "ctfe.pub",
                "ctfe_2022.pub",
                "fulcio.crt.pem",
                "fulcio_intermediate_v1.crt.pem",
                "fulcio_v1.crt.pem",
                "rekor.pub",
                "trusted_root.json"
        ]
}
```

create a key pair

```
cosign generate-key-pair
```

```
Enter password for private key: 
Enter password for private key again: 
Private key written to cosign.key
Public key written to cosign.pub
```

sign the policy image

```
cosign sign --key cosign.key ghcr.io/gertd/demo:0.0.0
```

```
Enter password for private key: 
WARNING: Image reference ghcr.io/gertd/demo:0.0.0 uses a tag, not a digest, to identify the image to sign.
    This can lead you to sign a different image than the intended one. Please use a
    digest (example.com/ubuntu@sha256:abc123...) rather than tag
    (example.com/ubuntu:latest) for the input to cosign. The ability to refer to
    images by tag will be removed in a future release.


        The sigstore service, hosted by sigstore a Series of LF Projects, LLC, is provided pursuant to the Hosted Project Tools Terms of Use, available at https://lfprojects.org/policies/hosted-project-tools-terms-of-use/.
        Note that if your submission includes personal data associated with this signed artifact, it will be part of an immutable record.
        This may include the email address associated with the account with which you authenticate your contractual Agreement.
        This information will be used for signing this artifact and will be stored in public transparency logs and cannot be removed later, and is subject to the Immutable Record notice at https://lfprojects.org/policies/hosted-project-tools-immutable-records/.

By typing 'y', you attest that (1) you are not submitting the personal data of any other person; and (2) you understand and agree to the statement and the Agreement terms at the URLs listed above.
Are you sure you would like to continue? [y/N] y
tlog entry created with index: 68494219
Pushing signature to: ghcr.io/gertd/demo
```

verify using the public key

```
cosign verify --key cosign.pub ghcr.io/gertd/demo:0.0.0
```

```
Verification for ghcr.io/gertd/demo:0.0.0 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key

[{"critical":{"identity":{"docker-reference":"ghcr.io/gertd/demo"},"image":{"docker-manifest-digest":"sha256:e583dc0e018fc11d659da7d1957c60ad8a523eafc142a9de391516eb45594e6f"},"type":"cosign container image signature"},"optional":{"Bundle":{"SignedEntryTimestamp":"MEUCIQDCaDwN4azHfMDAoSxPinbKVzBjC11VLz5H8M2SCJDQCAIgBjJs60kKyd++ih2N/Yv6UyHTOtWG71WbJKz6sq/aveM=","Payload":{"body":"eyJhcGlWZXJzaW9uIjoiMC4wLjEiLCJraW5kIjoiaGFzaGVkcmVrb3JkIiwic3BlYyI6eyJkYXRhIjp7Imhhc2giOnsiYWxnb3JpdGhtIjoic2hhMjU2IiwidmFsdWUiOiIxMTFmODZkYTZhOGNhZjBiN2JjM2U0YWI2Y2Q4YmNjNmQ4ZDUwZGM1MzI5NDM2YWY2ZmVhZmMwYjViODM1MjRiIn19LCJzaWduYXR1cmUiOnsiY29udGVudCI6Ik1FVUNJUUNzZHpycGxpZExjaCtNRW1qZEVhQ3JGTFJkUUl1WVBVTUc4QVJTdGpFbG9RSWdNUTNuaXJKODUybEFSU2ZnOGF4aHBuUEJURGh4Q1BtS004OHlYVCtoYytFPSIsInB1YmxpY0tleSI6eyJjb250ZW50IjoiTFMwdExTMUNSVWRKVGlCUVZVSk1TVU1nUzBWWkxTMHRMUzBLVFVacmQwVjNXVWhMYjFwSmVtb3dRMEZSV1VsTGIxcEplbW93UkVGUlkwUlJaMEZGY3pSQmFWbFJWWFkyTDJGNGJ6ZE5RVU5ZVDFWVmQwNW9Za2wwU2dvd2JEVmtkMmcxUldOTllrOVJibEJVWVdkclFUaHNTRzVNVGpKNldqaHNSMmd2ZVdoTk1XUTBTREZhUzFOeFJtaFhWMnRIZFRWdmJsbEJQVDBLTFMwdExTMUZUa1FnVUZWQ1RFbERJRXRGV1MwdExTMHRDZz09In19fX0=","integratedTime":1706849798,"logIndex":68494219,"logID":"c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d"}}}}]
```