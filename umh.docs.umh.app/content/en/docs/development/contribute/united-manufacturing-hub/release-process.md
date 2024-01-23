---
title: "Release Process"
content_type: task
description: |
    This page describes how to release a new version of the United Manufacturing
    Hub.
weight: 50
---

<!-- overview -->

Releases are coordinated by the United Manufacturing Hub team. All the features
and bug fixes due for a release are tracked in the internal project board.

Once all the features and bug fixes for a release are ready and merged into the
`staging` branch, the release process can start.

<!-- companion -->

## Companion
{{% notice note %}}
This section is for internal use at UMH.
{{% /notice %}}

### Testing

If a new version of the Companion is ready to be released, it must be tested
before it can be published. The testing process is done in the `staging`
environment.

The developer can push to the `staging` branch all the changes that needs to be
tested, including the new version definition in the Updater and in the
`version.json` file. They can then use the `make docker_tag
GIT_TAG=<semver-tag-to-be-released>` command from the Companion directory to
build and push the image. After that, from the staging environment, they can
trigger the update process.

{{% notice warning %}}
This process will not make the changes available to the user, but keep in mind
that the tagged version could still be accidentally used. Once the testing is
done, all the changes are pushed to `main` and the new release is published,
the image will be overwritten with the correct one.
{{% /notice %}}

### Preparing the Documentation

Begin by drafting new documentation within the `/docs/whatsnew` directory of the [United Manufacturing Hub documentation repository](https://github.com/united-manufacturing-hub/umh.docs.umh.app). Your draft should comprehensively include:
- The UMH version rolled out with this release.
- The new Companion version.
- Versions of any installed plugins, such as `Benthos-UMH`.

Initiate your document with an executive summary that encapsulates updates and changes across all platforms, including UMH and Companion.

### Version Update Procedure

Navigate to the [ManagementConsole repository](https://github.com/united-manufacturing-hub/ManagementConsole) and contribute a new `.go` file within the `/updater/cmd/upgrades` path. This file's name must adhere to the semantic versioning convention of the update (e.g., `0.0.5.go`).

This file should:
- Implement the `Version` interface defined in `upgrade_interface.go`.
-  Include PreMigration and PostMigration functions. These functions should return another function that, when executed, returns nil unless specific migration tasks are necessary. This nested function structure allows for conditional execution of migration steps, as demonstrated in the PostMigration example below:
   ```go
   func (v *v0x0x5) PostMigration() func(version *semver.Version, clientset kubernetes.Interface) error {
       return func(version *semver.Version, clientset kubernetes.Interface) error {
           zap.S().Infof("Post-Migration 0.0.5")
           return nil
       }
   }
   ```
- Define `GetImageVersion` to return the Docker tag associated with the new version. For `0.5.0` this would look like:
   ```go
   func (v *v0x0x5) GetImageVersion() *semver.Version {
       return semver.New(0, 0, 5, "", "")
   }
   ```
- Specify any Kubernetes controllers (e.g., Statefulsets, Deployments) needing restart post-update in the `GetPodControllers` function. Usually you just need to restart the companion itself, so you can use:
   ```go
   func (v *v0x0x5) GetPodControllers() []types.KubernetesController {
       return []types.KubernetesController{
           {
               Name: constants.StatefulsetName,
               Type: types.Statefulset,
           },
       }
   }
   ```

{{% notice note %}}
Validate that all kubernetes objects referenced here, are designed to restart after terminating their Pod.
This is especially important for Jobs.
{{% /notice %}}

Inside the `versions.go`, ensure to add your version inside the `buildVersionLinkedList` function.
```go
func buildVersionLinkedList() error {
	var err error
	builderOnce.Do(func() {
		zap.S().Infof("Building version list")
		start := v0x0x1{}
		versionLinkedList = &start
		/*
		    Other previous versions
		 */
		
		// Our new version
		err = addVersion(&v0x0x5{})
		if err != nil {
			zap.S().Warnf("Failed to add 0.0.5 to version list: %s", err)
			return
		}
		zap.S().Infof("Build version list")
	})
	return err
}
```

Update the `version.json` in the `frontend/static/version` directory with the new image tag and incorporate the changelog derived from your initial documentation draft.
```json
{
  "companion": {
    "versions": [
      {
        "semver": "0.0.1",
        "changelog": {
          "full": ["INTERNAL TESTING 0.0.1"],
          "short": "Bugfixes"
        },
        "requiresManualIntervention": false
      },
       
       // Other previous versions        

       // Our new version 
      {
        "semver": "0.0.5",
        "changelog": {
          "full": ["See 0.0.4"],
          "short": "This version is the same as 0.0.5 and is used for upgrade testing"
        },
        "requiresManualIntervention": false
      }
    ]
  }
}
```

### Finalizing the Release

To finalize:
1. Submit a PR to the [documentation repository](https://github.com/united-manufacturing-hub/umh.docs.umh.app) to transition the release notes from draft to final.
2. Initiate a PR from the staging to the main branch within the [ManagementConsole repository](https://github.com/united-manufacturing-hub/ManagementConsole), ensuring to reference the documentation PR.
3. Confirm the success of all test suites.
4. Merge the code changes and formalize the release on GitHub, labeling it with the semantic version (e.g., `0.0.5`, excluding any preceding `v`).
5. Merge the documentation PR to publicize the new version within the official documentation.


#### Checklist

- [ ] Draft documentation in `/docs/whatsnew` with version details and summary.
- [ ] Add new `.go` file for version update in `/updater/cmd/upgrades`.
- [ ] Implement `Version` interface and necessary migration functions.
- [ ] Update `version.json` with new image tag and changelog.
- [ ] Submit PR to finalize documentation.
- [ ] Create and merge PR in ManagementConsole repository, referencing documentation PR.
- [ ] Validate tests and merge code changes.
- [ ] Release new GitHub version without the `v` prefix.
- [ ] Merge documentation PR to publish new version details.



<!--
   In the future we might need to add some additional steps, if the companion also updates the UMH
-->

<!-- helm -->

## Helm Chart

### Prerelease

The prerelease process is used to test the release before it is published.
If bugs are found during the prerelease, they can be fixed and the release
process can be restarted. Once the prerelease is finished, the release can be
published.

1. Create a prerelease branch from `staging`:

    ```bash
    git checkout staging
    git pull
    git checkout -b <next-version>-prerelease1
    ```

2. Update the `version` and `appVersion` fields in the `Chart.yaml` file to the
   next version:

    ```yaml
    version: <next-version>-prerelease1
    appVersion: <next-version>-prerelease1
    ```

3. Navigate to the `deployment/helm-repo` directory and run the following
   commands:

    ```bash
    helm package ../united-manufacturing-hub
    helm repo index --url https://staging.united-manufacturing-hub.pages.dev --merge index.yaml .
    ```

   Pay attantion to use `-` instead of `.` as a separator in `<next-version>`.

4. Commit and push the changes:

    ```bash
    git add .
    git commit -m "build: <next-version>-prerelease1"
    git push origin <next-version>-prerelease1
    ```

5. Merge prerelease branch into `staging`

### Test

All the new releases must be thoroughly tested before they can be published.
This includes specific tests for the new features and bug fixes, as well as
general tests for the whole stack.

General tests include, but are not limited to:

- Deploy the stack with flatcar
- Upgrade the stack from the previous version
- Deploy the stack on Karbon 300 and test with real sensors

If any bugs are found during the testing phase, they must be fixed and pushed
to the prerelease branch. Multiple prerelease versions can be created if
necessary.

### Release

Once all the tests have passed, the release can be published. Merge the
prerelease branch into `staging` and create a new release branch.

1. Create a release branch from `staging`:

   ```bash
   git checkout main
   git pull
   git checkout -b <next-version>
   ```

2. Update the `version` and `appVersion` fields in the `Chart.yaml` file to the
   next version:

   ```yaml
   version: <next-version>
   appVersion: <next-version>
   ```

3. Navigate to the `deployment/helm-repo` directory and run the following
   commands:

   ```bash
   helm package ../united-manufacturing-hub
   helm repo index --url https://repo.umh.app --merge index.yaml .
   ```

4. Commit and push the changes, tagging the release:

     ```bash
     git add .
     git commit -m "build: <next-version>"
     git tag <next-version>
     git push origin <next-version> --tags
     ```

5. Merge the release branch into `staging`

6. Merge `staging` into `main` and create a new release from the tag on
   GitHub.
