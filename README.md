# Package for kpack dependencies

### Create a new deps package
 
1. Create a new values file:
   ```yaml
   #@data/values
   ---
   store_images:
     - image: "gcr.io/paketo-buildpacks/java"
     - image: "gcr.io/paketo-buildpacks/nodejs"
   
   stacks:
     - name: "base"
       build:
         image: "paketobuildpacks/build:base-cnb"
       run
         image: "paketobuildpacks/run:base-cnb"
   
   builders:
     - name: "base"
       stack: "base"
       order:
         - group:
             - id: "paketo-buildpacks/java"
         - group:
             - id: "paketo-buildpacks/nodejs"
   ```
2. Create package dir

   ```bash
   mkdir -p /tmp/package
   mkdir -p /tmp/package/.imgpkg
   ```
   
3. Assemble package
   
   ```bash
   cp -r config /tmp/package
   cp <values file> /tmp/package/config
   kbld -f <values file> --imgpkg-lock-output /tmp/package/.imgpkg/images.yml 
   ```
   
4. Push package image
   
   ```bash
   imgpkg push -b <repo> -f /tmp/package --lock-output pushed_bundle.lock
   ```
   
5. Copy image reference from pushed_bundle.lock and use it to template package

   ```bash
   ytt -f package.yaml -v bundle_image=<pushed bundle ref> -v version="0.0.0" > templated-package.yaml
   ```

### Install the deps package
   
0. prerequisite: install kapp-controller

   ```bash
   kapp deploy -a kc -f https://github.com/vmware-tanzu/carvel-kapp-controller/releases/latest/download/release.yml
   ```

1. Apply the templated package from step #5 above along with the metadata.yaml
   
   ```bash
   kapp deploy -a kpack-deps-package -f templated-package.yaml -f metadata.yaml
   ```
2. Create a file defining a secret containing the user provided values
   
   ```yaml
   ---
   apiVersion: v1
   kind: Secret
   metadata:
    name: kpack-deps-values
    namespace: default
   stringData:
    values.yaml: |
      ---
      repository: <writeable repo for deps>
      repsitory_secret:
        name: <name of secret for repo>
        namespace: <namespace of secret>
   ```
   or
   ```yaml
   ---
   apiVersion: v1
   kind: Secret
   metadata:
    name: kpack-deps-values
    namespace: default
   stringData:
    values.yaml: |
      ---
      repository: <writeable repo for deps>
      repsitory_username: "username"
      repsitory_password: "password"
   ```
   
3. Create a file defining the package install

   ```yaml
   ---
   apiVersion: packaging.carvel.dev/v1alpha1
   kind: PackageInstall
   metadata:
     name: kpack-deps
     namespace: default
   annotations:
     kapp.k14s.io/change-rule: "delete before deleting serviceaccount"
   spec:
     serviceAccountName: <service account with admin privilege>
     packageRef:
       refName: kpack-dependencies.community.tanzu.vmware.com
       versionSelection:
         constraints: 0.0.0
     values:
     - secretRef:
         name: kpack-deps-values
   ```
4. Deploy the secret and package install

   ```bash
   kapp deploy -a kpack-deps-install -f <values secret file> -f <package install file>
   ```