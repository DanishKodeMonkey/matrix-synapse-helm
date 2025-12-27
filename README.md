# matrix-synapse-helm

Work in progress

A k3s based helm chart for deploying a matrix-synapse serveer on a homelab. 

The intend for this is to provide a relatively easy starting point to winding up and having a ready-to-use matrix server to use as you will and expand upon.

What the helm will do:

The helm will help provide the following:
The deploymeent of the matrix-synapsee server, with some baselinee configuration
The deployment of a element frontend for your users to register on.
Persistent volume claims for the persistenece of data, media, users, etc.
Ingresses for traffic to and from the matrix server, and element frontend.

What is recommended outside of this is:
A dedicated namespace
CRDs
Ingress controllers

All of which should be set up to accomidate the deployment.


The PVC is needed for synapse for:
Databasee files (users)
media uploads (from users)
signing keys 
state
configuration

all of this lives in a /data volume.

If the /data volume dissapears, users, federations, and server identity may be lost. 

to make a admin, use the official matrix user command

``` bash
kubectl exec -n matrix -it <your pod> -- \
  register_new_matrix_user \
  -c /data/homeserver.yaml \
  --user <username> \
  --password <password> \
  --admin
```

load secrets from non-commited .secrets.yaml file along with rest of values.yaml
```bash
helm upgrade --install matrix ./ -n matrix -f values.yaml -f .secrets.yaml      
```

The following is needed in .secrets.yaml
```bash
synapse:
  recaptcha:
    siteKey: "recaptcha site key"
    secretKey: "repatchasecreetkey
  registration_shared_secret: "some secret shared secret"
```