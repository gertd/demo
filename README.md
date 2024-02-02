Open Policy Container demo steps

login to the container registry

```
gh auth token | policy login -s ghcr.io -u $USER --password-stdin

Logging in.
server: ghcr.io
user: gertd

OK.
```

lets create a policy to play with

list available templates

```
policy templates list

Fetching templates 

  NAME             KIND    DESCRIPTION                    
  github           cicd    GitHub policy CI/CD template.  
  gitlab           cicd    GitLab policy CI/CD template.  
  policy-template  policy  Minimal policy template. 
```

lets create a policy

```
policy templates apply policy-template

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

Created new image.
digest: sha256:0f66afb3eaafbb9056a1bc1e5bc90ecdf6bcbdf9288f6e2e398cd3c23ae71a83

Tagging image.
reference: ghcr.io/demo-policy:0.0.0
```

where did it go? 
```
policy images

  REPOSITORY   TAG    IMAGE ID      CREATED               SIZE  
  demo-policy  0.0.0  e583dc0e018f  2024-02-02T04:13:29Z  736B  
```

lets load the policy from the local store and run it using policy repl

```
policy repl ghcr.io/demo-policy:0.0.0
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
