# OpenShift Console Plugin for Tekton CD / OpenShift Pipelines

:construction: This is a WIP [dynamic plugin](https://github.com/openshift/console/tree/master/frontend/packages/console-dynamic-plugin-sdk) that extends the [OpenShift Console](https://github.com/openshift/console) by adding **Tekton CD / OpenShift Pipelines** specific features.

It will be shipped and enabled automatically by the [Tekton CD / OpenShift Pipelines operator](https://github.com/tektoncd/operator).

## Rough roadmap

* 2023 / v1: Adds a new **Dashboard** and **Metrics** tab to the Pipeline pages that shows aggregated PipelineRun stats from the [Tekton Results API](https://github.com/tektoncd/results)
* 2024 / v2: Additional pages, like the Pipeline list and details pages are moved from the "static" [pipelines-plugin](https://github.com/openshift/console/tree/master/frontend/packages/pipelines-plugin) from the OpenShift Console into this project.

## Compatiblity

This initial version of this plugin will be shipped with OpenShift Pipelines operator 1.1x and requires at least OpenShift 4.12.

See also the OpenShift Pipelines [Compatibility and support matrix](https://docs.openshift.com/pipelines/latest/about/op-release-notes.html#compatibility-support-matrix_op-release-notes)

## Development

### Getting started

Required tools:

* [Node.js](https://nodejs.org/en/) v16 or newer and [yarn](https://yarnpkg.com) are required
to build and run the example.
* To run OpenShift console in a container, either
[Docker](https://www.docker.com) or [podman 3.2.0+](https://podman.io) and
[oc](https://console.redhat.com/openshift/downloads) are required.

You can run the plugin using a local development environment or build an image
to deploy it to a cluster.


### Option 1: Local development, with a local (cloned) console

This option is a good candidate if you build and run the OpenShift Console already on your machine.

In one terminal window, run:

```
cd console-plugin
yarn install
yarn start # or yarn dev
```

In another terminal window, run:

```
cd console
# the first time you need to build the backend and frontend with ./build.sh
# oc login...
# source ./contrib/oc-environment.sh
./bin/bridge -plugins "pipeline-console-plugin=http://localhost:9001"
```

### Option 2: Local development, running a console as container

In one terminal window, run:

1. `yarn install`
2. `yarn start`

In another terminal window, run:

1. `oc login` (requires [oc](https://console.redhat.com/openshift/downloads) and an [OpenShift cluster](https://console.redhat.com/openshift/create))
2. `yarn run start-console` (requires [Docker](https://www.docker.com) or [podman 3.2.0+](https://podman.io))

This will run the OpenShift console in a container connected to the cluster
you've logged into. The plugin HTTP server runs on port 9001 with CORS enabled.
Navigate to <http://localhost:9000/example> to see the running plugin.

#### Running start-console with Apple silicon and podman

If you are using podman on a Mac with Apple silicon, `yarn run start-console`
might fail since it runs an amd64 image. You can workaround the problem with
[qemu-user-static](https://github.com/multiarch/qemu-user-static) by running
these commands:

```bash
podman machine ssh
sudo -i
rpm-ostree install qemu-user-static
systemctl reboot
```

### Option 3: Docker + VSCode Remote Container

Make sure the
[Remote Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
extension is installed. This method uses Docker Compose where one container is
the OpenShift console and the second container is the plugin. It requires that
you have access to an existing OpenShift cluster. After the initial build, the
cached containers will help you start developing in seconds.

1. Create a `dev.env` file inside the `.devcontainer` folder with the correct values for your cluster:

```bash
OC_PLUGIN_NAME=pipeline-console-plugin
OC_URL=https://api.example.com:6443
OC_USER=kubeadmin
OC_PASS=<password>
```

2. `(Ctrl+Shift+P) => Remote Containers: Open Folder in Container...`
3. `yarn start`
4. Navigate to <http://localhost:9000/example>

## Container image

Before you can deploy your plugin on a cluster, you must build an image and
push it to an image registry.

1. Build the image:

   ```sh
   docker build -t quay.io/my-repository/my-plugin:latest .
   ```

2. Run the image:

   ```sh
   docker run -it --rm -d -p 9001:80 quay.io/my-repository/my-plugin:latest
   ```

3. Push the image:

   ```sh
   docker push quay.io/my-repository/my-plugin:latest
   ```

NOTE: If you have a Mac with Apple silicon, you will need to add the flag
`--platform=linux/amd64` when building the image to target the correct platform
to run in-cluster.

## Deployment on cluster

A [Helm](https://helm.sh) chart is available to deploy the plugin to an OpenShift environment.

The following Helm parameters are required:

`plugin.image`: The location of the image containing the plugin that was previously pushed

Additional parameters can be specified if desired. Consult the chart [values](charts/openshift-console-plugin/values.yaml) file for the full set of supported parameters.

### Installing the Helm Chart

Install the chart using the name of the plugin as the Helm release name into a new namespace or an existing namespace as specified by the `plugin_console-plugin-template` parameter and providing the location of the image within the `plugin.image` parameter by using the following command:

```shell
helm upgrade -i  my-plugin charts/openshift-console-plugin -n plugin__console-plugin-template --create-namespace --set plugin.image=my-plugin-image-location
```

NOTE: When deploying on OpenShift 4.10, it is recommended to add the parameter `--set plugin.securityContext.enabled=false` which will omit configurations related to Pod Security.

NOTE: When defining i18n namespace, adhere `plugin__<name-of-the-plugin>` format. The name of the plugin should be extracted from the `consolePlugin` declaration within the [package.json](package.json) file.

## i18n

The plugin use [react-i18next](https://react.i18next.com/) to translate messages.
The i18n namespace must match the name of the `ConsolePlugin` resource with the `plugin__` prefix to avoid
naming conflicts. For this plugin this means `plugin__pipeline-console-plugin`.

All translation calls like the `useTranslation` hook must use this namespace as follows:

```tsx
conster Header: React.FC = () => {
  const { t } = useTranslation('plugin__pipeline-console-plugin');
  return <h1>{t('Hello, World!')}</h1>;
};
```

For labels in `console-extensions.json`, you can use the format
`%plugin__pipeline-console-plugin~My Label%`. Console will replace the value with
the message for the current language from the `plugin__pipeline-console-plugin`
namespace. For example:

```json
  {
    "type": "console.navigation/href",
    "properties": {
      "id": "pipelines-overview",
      "perspective": "admin",
      "section": "pipelines",
      "name": "%plugin__pipeline-console-plugin~Overview%"
    }
  }
```

Running `yarn i18n` updates the JSON files in the `locales` folder of the
plugin when adding or changing messages.

## Linting

This project adds prettier, eslint, and stylelint. Linting can be run with
`yarn run lint`.

The stylelint config disallows hex colors since these cause problems with dark
mode (starting in OpenShift console 4.11). You should use the
[PatternFly global CSS variables](https://patternfly-react-main.surge.sh/developer-resources/global-css-variables#global-css-variables)
for colors instead.

The stylelint config also disallows naked element selectors like `table` and
`.pf-` or `.co-` prefixed classes. This prevents plugins from accidentally
overwriting default console styles, breaking the layout of existing pages. The
best practice is to prefix your CSS classnames with your plugin name to avoid
conflicts. Please don't disable these rules without understanding how they can
break console styles!

## Reporting

Steps to generate reports

1. In command prompt, navigate to root folder and execute the command `yarn run cypress-merge`
2. Then execute command `yarn run cypress-generate`
The cypress-report.html file is generated and should be in (/integration-tests/screenshots) directory

## References

- [Console Plugin SDK README](https://github.com/openshift/console/tree/master/frontend/packages/console-dynamic-plugin-sdk)
- [Customization Plugin Example](https://github.com/spadgett/console-customization-plugin)
- [Dynamic Plugin Enhancement Proposal](https://github.com/openshift/enhancements/blob/master/enhancements/console/dynamic-plugins.md)
