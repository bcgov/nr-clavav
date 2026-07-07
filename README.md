## Cloned from https://github.com/bcgov/clamav with edits to the code to match our needs.

ClamAV® is an open source antivirus engine for detecting trojans, viruses, malware & other malicious threats.

This is a repo setup for utilization in Red Hat Openshift. This solution allows you to create a pod in your openshift environment to scan any file for known virus signatures, quickly and effectively.

The builds package the barebones service, and the deployment config will download latest signatures on first run.

Freshclam can be run within the container at any time to update the existing signatures. Alternatively, you can re-deploy which will fetch the latest into the running container.

# Deployment

The templates in the [openshift/templates](./openshift/templates) will build and deploy the app. Modify to suit your own environment. [openshift/templates/clamav-bc.conf](./openshift/templates/clamav-bc.conf) will create your builder image (ideally in your tools project), and [openshift/templates/clamav-dc.conf](./openshift/templates/clamav-dc.conf) will create the pod deployment. Modify the environment variables defined in both the build config and deployment config appropriately.

Prerequisites (needed for both steps below):

- helm client installed https://helm.sh/docs/intro/install/
- oc cli client installed (you should be able to find this on openshift under command line tools) https://console.apps.silver.devops.gov.bc.ca/command-line-tools
- logged into openshift from the CLI:

```
oc login --web
OR
copy paste oc login token from openshift
```

> **Note:** Replace `<tools-namespace>` below with your OpenShift tools namespace (e.g. `<licenseplate>-tools`). This should match `clamav.imageNamespace` in the Helm values.

## Step 1 — Build and tag the image

**Do this before deploying the chart.** The Helm chart only *pulls* an image from the internal registry — it does **not** build one. The image (at the version the chart expects) must already exist in your tools namespace, or the pods will fail with `ImagePullBackOff`. Run this the first time, and again whenever you change the ClamAV version.

The chart pulls the tag matching `appVersion` in [charts/clamav/Chart.yaml](./charts/clamav/Chart.yaml) (currently **1.0.5**), so tag the image to that same version.

1. **CLI** (first time only) create the ImageStream + BuildConfig in your tools namespace:

```
oc process -f openshift/templates/clamav-bc.conf | oc apply -n <tools-namespace> -f -
```

2. Run the build. This produces `clamav:latest`.

   - **WEB**: run the build config on openshift https://console.apps.silver.devops.gov.bc.ca/k8s/ns/<tools-namespace>/build.openshift.io~v1~BuildConfig?name=clamav-build
     ![alt text](image.png)
   - **CLI**: `oc start-build clamav-build -n <tools-namespace> --follow`

3. **CLI** tag `latest` to the version the chart expects (must match `appVersion`, e.g. 1.0.5):

```
oc tag <tools-namespace>/clamav:latest <tools-namespace>/clamav:1.0.5
```

4. **CLI** confirm the versioned tag now exists before deploying:

```
oc get istag -n <tools-namespace> | grep clamav
```

## Step 2 — Deploy with Helm

The Helm chart deploys a standard StatefulSet. This deployment should work on [OpenShift Local](https://github.com/crc-org/crc), [kind](https://kind.sigs.k8s.io/) or even [Docker Desktop](https://docs.docker.com/desktop/kubernetes/)

1. **CLI** Navigate to the clamav folder in this repository

2. **EDITOR** Set the image namespace. `clamav.imageNamespace` is **required** — the chart will fail to render if it is not supplied. Set it either in the `<env>/values.yaml` file or on the command line with `--set` (see step 3). Below is the image snippet from the values.yaml file. Leave `tag` empty to inherit `appVersion` from Chart.yaml (recommended), or set it to pin a specific version.

```
clamav:
  # clamav.imageRegistry -- The registry hosting the clamav docker image
  imageRegistry: image-registry.openshift-image-registry.svc:5000
  # clamav.imageNamespace -- (required) The OpenShift tools namespace hosting the image, e.g. <licenseplate>-tools
  imageNamespace: <tools-namespace>
  # clamav.imageName -- The name of the clamav image within the namespace
  imageName: clamav
  # clamav.version -- defaults to .Chart.appVersion when left empty
  tag: ""
```

3. **CLI** run helm upgrade command in the proper namespace **make sure you are in the proper location in the clam av folder**. Pass the image namespace with `--set clamav.imageNamespace=...` (unless you already set it in the env values file).

template

```
helm upgrade --install -n <namespace>-<env> clamav ./charts/clamav -f ./charts/clamav/<env>/values.yaml --set clamav.imageNamespace=<tools-namespace>
```

example

```
helm upgrade --install -n <yournamespace> clamav ./charts/clamav -f ./charts/clamav/dev/values.yaml --set clamav.imageNamespace=<tools-namespace>
```

## Accessing the service

clamd speaks the raw TCP clamd protocol (not HTTP) on port **3310**. On OpenShift, reach it in-cluster through the `Service` — no Route is used. Other pods connect to:

```
<release>-clamav.<namespace>.svc:3310
# within the same namespace, simply: <release>-clamav:3310
```

### Running / testing locally

You can talk to clamd from your workstation without OpenShift in a couple of ways.

**Option A — run the image directly with Docker:**

```
docker build -t clamav:local .
docker run --rm -p 3310:3310 clamav:local
```

**Option B — port-forward an existing deployment** (works on OpenShift, kind, or Docker Desktop Kubernetes):

```
oc port-forward svc/<release>-clamav 3310:3310
# or: kubectl port-forward svc/<release>-clamav 3310:3310
```

Either way clamd is then reachable on `localhost:3310`. Test it with `clamdscan`, or with a raw ping:

```
# expects: PONG
echo -e 'zPING\0' | nc -q1 localhost 3310
```

To scan a file locally, point `clamdscan` at the forwarded port:

```
clamdscan --config-file=/dev/stdin somefile <<'EOF'
TCPSocket 3310
TCPAddr localhost
EOF
```

> **Note:** clamd needs its signature database before it will answer scans — on first start the container runs `freshclam`, which can take a minute or two. The startup probe already accounts for this in-cluster.

## How to upgrade the ClamAV version

1. Update `appVersion` in [charts/clamav/Chart.yaml](./charts/clamav/Chart.yaml) to the new version.
2. Rebuild and re-tag the image to the new version — repeat **Step 1** above (the tag in step 3 must match the new `appVersion`).
3. Redeploy — repeat **Step 2** above.

## Issues

- if the clamAV version is greater than 1.0.6 we get an error in the docker image. source: https://github.com/Cisco-Talos/clamav/issues/1371 I ended up just using the 1.0.5 image and tested with a fake virus file from https://www.eicar.org/download-anti-malware-testfile/ to confirm our image still works.
