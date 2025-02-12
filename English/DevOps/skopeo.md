Also, translate this:

## Skopeo

Skopeo is a simple command line tool to interact with registries and manage — inspect, copy and delete — container images. While these tasks can also be done with Podman, Skopeo is a more lightweight “do one thing and do it well” kind of tool.

The binary can be easily installed on macOS with brew install skopeo and we don’t need a Linux host at all — feel free to stop your Linux VM.

```
$ skopeo -v
skopeo version 0.1.40
```

Before we can do anything useful, we need to create a containers-policy file that contains verification rules to decide whether we accept a pulled image or not. The default location is/etc/containers/policy.json or we can point to another file adding the --policy <policy file> parameter. Alternatively we can use the --insecure-policy parameter to accept all images. Note, these policy parameters are “global” parameters and must be added before the action command (i.e. inspect, copy, delete, etc). The easiest is to create a policy file allowing all registries at the default location:

```
$ sudo sh -c 'cat >/etc/containers/policy.json' <<EOF 
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
                }
        }
}
EOF
```

To access registries requiring authentication we need to pass credentials somehow. We have two options:

- username:password: Using the `--creds`, `--src-creds`, `--dest-creds` parameters based on the command
- authfile: Docker auth file which is the same as above for Podman. It can be added by `--authfile <auth file>`. If it’s not set, Skopeo looks at the default location: `$HOME/.docker/config.json`. Here is an example file:

```json
{
 "auths": {
  "registry.redhat.io": {
    "auth": "<username:password in base64>"
  },
  "quay.io": {
    "auth": "bXl1c2VyOm15c2VjcmV0"
  }
 }
}
```

Trusting or ignoring TLS certificates can be achieved by a variety of parameters based on the Skopeo command being used: `--cert-dir`, `--tls-verify`, `--src-cert-dir`, `--src-tl-verify`, `--dest-cert-dir`, `--dest-tls-verify`.

### Inspect

Skopeo can be used to read information about an image from a registry — without having to download the image first:

```
$ skopeo inspect docker://quay.io/quay/busybox:latest
{
    "Digest": "sha256:91aa1e3e55765a568...",
    "RepoTags": ...
```

With registries supporting multiple OS versions of an image adding the global parameter--override-os linux is required otherwise it defaults to MacOS OS darwin that doesn’t exist.

```
$ skopeo --override-os linux inspect docker://docker.io/nginx$ skopeo --override-os linux inspect --authfile myauth.json docker://registry.redhat.io/openshift3/jenkins-2-rhel7
```

For detailed information (e.g. layers, entrypoint, cmd) check the raw response from the registry:

```
$ skopeo --override-os linux inspect --config --raw docker://docker.io/nginx | jq .
```

Inspect can also be used on images saved on the local disk in some format (e.g. if the image has been downloaded usingskopeo copy):

```
# Image saved as docker-archive
$ skopeo inspect docker-archive:/tmp/mytag.latest.tar# Inspecting oci-archive images tries to use /var/tmp/, needs sudo
$ sudo skopeo inspect oci-archive:/tmp/mytag.latest.oci# Image saved as directory or oci directory
$ skopeo inspect dir:/tmp/mytag-latest-dir/
$ skopeo inspect oci:/tmp/mytag-latest-oci/
```


### Copy

The most common task we’ll do with Skopeo is to copy images from one registry to another. There are several parameters needed, but they are pretty obvious after understanding the basic parameters we used with the inspect commands above. In the following example we will copy a Jenkins image from `registry.redhat.io` to `quay.io`:

```
# Simple case - policy and auth file prepared$ skopeo --override-os linux copy docker://registry.redhat.io/openshift3/jenkins-2-rhel7:v3.11 docker://quay.io/bszeti/jenkins:v3.11
```

Note: `--override-os` linux may or may not be needed based on the registry just like for skopeo inspect

We can now layer in the authentication and TLS parameters previously discussed for example to copy an image between private repositories:

```
$ skopeo --insecure-policy copy 
--src-tls-verify=false --dest-tls-verify=false 
--src-creds 'devuser:secret' --dest-creds 'myuser:secret' docker://devregistry.host:5000/myteam/myapp:v1 docker://myregistry.host:8443/myteam/myapp:mytag
```

We can also use `skopeo copy` to tag images. Skopeo checks if the layers already exist in the destination registry, so no data is copied over unnecessarily.

```
$ skopeo copy
docker://myregistry:8443/myteam/myapp:v1 docker://myregistry:8443/myteam/myapp:qa
```

It’s also possible to use the copy command to save images on the local disk in different formats.

```
$ skopeo copy docker://docker.io/nginx docker-archive:/tmp/nginx.tar
$ skopeo copy docker://docker.io/nginx oci-archive:/tmp/nginx.oci
$ skopeo copy docker://docker.io/nginx dir:/tmp/nginx-dir
$ skopeo copy docker://docker.io/nginx oci:/tmp/nginx-oci
```

Note: In some cases Skopeo uses temporary directories under `/var/tmp` when managing archives, so we may run into permission issues unless it’s run as root.

#### Summary

The new set of daemonless, open-source container management tools have reached a maturity level when they can confidently be used on a Linux box. Unfortunately they are not yet fully native with macOS. However, with some manual preparation and some minor caveats, we can begin using these tools now. Hopefully this guide has helped you to make that journey less complicated while the ongoing work brings us towards a more seamless experience.


Source:


https://itnext.io/podman-and-skopeo-on-macos-1b3b9cf21e60