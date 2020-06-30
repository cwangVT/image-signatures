# image-signatures
This repo is to store image signatures for quay.io and docker.io images

## Steps to signing images

1. setup key fingureprint with `gpg2`

2. sign and push image:
```
sudo skopeo copy \
--src-creds "${SRC_REGISTRY_USER}:${SRC_REGISTRY_TOKEN}"  \
--dest-creds "${DEST_REGISTRY_USER}:${DEST_REGISTRY_TOKEN}"  \
--remove-signatures  \
--sign-by ${PUBKEY_FINGERPRINT}  \
docker://${SRC_IMAGE} \
docker://${DEST_IMAGE}-signed

```

3. upload the signature in `/var/lib/atomic/sigstore/` to docker.io or quay.io folder


## Steps to verify image with signature (with skopeo)

1. set `/etc/containers/registry.d/default.yaml` to
```
default-docker:
  # sigstore: file:///var/lib/atomic/sigstore
 
  sigstore-staging: file:///var/lib/atomic/sigstore

docker:
    quay.io:
      sigstore: https://raw.githubusercontent.com/cwangVT/image-signatures/master/quay.io
    docker.io:
      sigstore: https://raw.githubusercontent.com/cwangVT/image-signatures/master/docker.io

```
2. set `/etc/containers/policy.json` to
```
{
    "default": [
        {
            "type": "insecureAcceptAnything"
        }
    ],
    "transports":
        {
            "docker-daemon":
                {
                    "": [{"type":"insecureAcceptAnything"}]
                },
            "docker":
                {
                    "quay.io": [
                        {
                            "type": "signedBy",
                            "keyType": "GPGKeys",
                            "keyPath": "/etc/pki/developers/public.gpg"
                        }
                    ],
                    "docker.io": [
                        {
                            "type": "signedBy",
                            "keyType": "GPGKeys",
                            "keyPath": "/etc/pki/developers/public.gpg"
                        }
                    ]

                }
        }
}
```
3. copy public key to `/etc/pki/developers/public.gpg`

4. run the command below to verify the signature, and only signed image can be copyed
```
skopeo copy \
--src-creds "${DEST_REGISTRY_USER}:${DEST_REGISTRY_TOKEN}"  \
docker://${DEST_IMAGE}-signed dir:tmp
```

## Steps to verify image with signature (with OCP)

1. `oc apply -f ocp-verify-test/signaturemc.yaml`. 

This will setup the policy.json, default.yaml, and public.gpg on all OCP nodes

2. create testing pods with yaml files in `ocp-verify-test` folder

Only the pods with signed images should become running. The other pods should failed with `ImagePullBackOff`
