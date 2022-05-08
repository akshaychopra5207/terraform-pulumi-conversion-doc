
### Prerequisites

Ensure the following tools are installed and present in your `$PATH`:

* [`pulumictl`](https://github.com/pulumi/pulumictl#installation)
* [Go 1.17](https://golang.org/dl/) or 1.latest
* [NodeJS](https://nodejs.org/en/) 14.x.  We recommend using [nvm](https://github.com/nvm-sh/nvm) to manage NodeJS installations.
* [Yarn](https://yarnpkg.com/)
* [TypeScript](https://www.typescriptlang.org/)
* [Python](https://www.python.org/downloads/) (called as `python3`).  For recent versions of MacOS, the system-installed version is fine.
* [.NET](https://dotnet.microsoft.com/download)

### Creating and Initializing the Repository

Pulumi offers this repository as a [GitHub template repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template) for convenience.  From this repository:

1. Clone the  repository pulumi-oneandone.

From the repository:

1. Run the following command to update files to use the name of your provider (third-party: use your GitHub organization/username):

    ```bash
    make prepare NAME=oneandone REPOSITORY=github.com/pulumi/pulumi-oneandone
    ```

   This will do the following:
   - rename folders in `provider/cmd` to `pulumi-resource-foo` and `pulumi-tfgen-foo`
   - replace dependencies in `provider/go.mod` to reflect your repository name
   - find and replace all instances of the boilerplate `xyz` with the `NAME` of your provider.

   Note for third-party providers:
   - Make sure to set the correct GitHub organization/username in all files referencing your provider as a dependency:
     - `examples/go.mod`
     - `provider/resources.go`
     - `sdk/go.mod`
     - `provider/cmd/pulumi-resource-foo/main.go`
     - `provider/cmd/pulumi-tfgen-foo/main.go`


### Composing the Provider Code - Prerequisites

Pulumi provider repositories have the following general structure:

* `examples/` contains sample code which may optionally be included as integration tests to be run as part of a CI/CD pipeline.
* `provider/` contains the Go code used to create the provider as well as generate the SDKs in the various languages that Pulumi supports.
  * `provider/cmd/pulumi-tfgen-foo` generates the Pulumi resource schema (`schema.json`), based on the Terraform provider's resources.
  * `provider/cmd/pulumi-resource-foo` generates the SDKs in all supported languages from the schema, placing them in the `sdk/` folder.
  * `provider/pkg/resources.go` is the location where we will define the Terraform-to-Pulumi mappings for resources.
* `sdk/` contains the generated SDK code for each of the language platforms that Pulumi supports, with each supported platform in a separate subfolder.

1. In `provider/go.mod`, add a reference to the upstream Terraform provider in the `require` section, e.g.

    ```go
    github.com/foo/terraform-provider-oneandone v0.0.5
    ```

1. In `provider/resources.go`, ensure the reference in the `import` section uses the correct Go module path, e.g.:

    ```go
    github.com/foo/terraform-provider-oneandone/oneandone
    ```

1. Download the dependencies:

    ```bash
    cd provider && go mod tidy && cd -
    ```

1. Create the schema by running the following command:

    ```bash
    make tfgen
    ```

    Note warnings about unmapped resources and data sources in the command's output.  We map these in the next section, e.g.:

    ```text
    warning: resource oneandone_something not found in provider map; skipping
    warning: resource oneandone_something_else not found in provider map; skipping
    warning: data source oneandone_something not found in provider map; skipping
    warning: data source oneandone_something_else not found in provider map; skipping
    ```

## Adding Mappings, Building the Provider and SDKs

In this section we will add the mappings that allow the interoperation between the Pulumi provider and the Terraform provider.  Terraform resources map to an identically named concept in Pulumi.  Terraform data sources map to plain old functions in your supported programming language of choice.  Pulumi also allows provider functions and resources to be grouped into _namespaces_ to improve the cohesion of a provider's code, thereby making it easier for developers to use.  If your provider has a large number of resources, consider using namespaces to improve usability.

The following instructions all pertain to `provider/resources.go`, in the section of the code where we construct a `tfbridge.ProviderInfo` object:

1. **Add resource mappings:** For each resource in the provider, add an entry in the `Resources` property of the `tfbridge.ProviderInfo`, e.g.:

    ```go
    // Most providers will have all resources (and data sources) in the main module.
    // Note the mapping from snake_case HCL naming conventions to UpperCamelCase Pulumi SDK naming conventions.
    // The name of the provider is omitted from the mapped name due to the presence of namespaces in all supported Pulumi languages.
    "oneandone_something":      {Tok: tfbridge.MakeResource(mainPkg, mainMod, "Something")},
    "oneandone_something_else": {Tok: tfbridge.MakeResource(mainPkg, mainMod, "SomethingElse")},
    ```
   
   [See the underlying terraform-bridge code here.](https://github.com/pulumi/pulumi-terraform-bridge/blob/master/pkg/tfbridge/info.go#L168)
1. **Add data source mappings:** For each data source in the provider, add an entry in the `DataSources` property of the `tfbridge.ProviderInfo`, e.g.:

    ```go
    // Note the 'get' prefix for data sources
    "oneandone_something":      {Tok: tfbridge.MakeDataSource(mainPkg, mainMod, "getSomething")},
    "oneandone_something_else": {Tok: tfbridge.MakeDataSource(mainPkg, mainMod, "getSomethingElse")},
    ```

1. **Add documentation mapping (sometimes needed):**  If the upstream provider's repo is not a part of the `terraform-providers` GitHub organization, specify the `GitHubOrg` property of `tfbridge.ProviderInfo` to ensure that documentation is picked up by the codegen process, and that attribution for the upstream provider is correct, e.g.:

    ```go
    GitHubOrg: "akshaychopra5207",
    ```

1. Build the provider binary and ensure there are no warnings about unmapped resources and no warnings about unmapped data sources:

    ```bash
    make provider
    ```

    You may see warnings about documentation and examples, including "unexpected code snippets".  These can be safely ignored for now.  Pulumi will add additional documentation on mapping docs in a future revision of this guide.

1. Build the SDKs in the various languages Pulumi supports:

    ```bash
    make build_sdks
    ```

**Note:** If you make revisions to code in `resources.go`, you must re-run the `make tfgen` target to regenerate the schema.
The `make tfgen` target will take the file `schema.json` and serialize it to a byte array so that it can be included in the build output.
(This is a holdover from Go 1.16, which does not have the ability to directly embed text files. We are working on removing the need for this step.)



### Terraform provider Upgradtion
    Understanding the upgrade process by reading terraform documenation and 
    Change the go files dependencies to "github.com/hashicorp/terraform-plugin-sdk/v2"

