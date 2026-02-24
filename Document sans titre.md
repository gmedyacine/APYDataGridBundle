# **CI/CD Pipeline Documentation: Publishing Artifacts to JFrog Artifactory**

## **1\. Overview and Objective**

This document outlines the process of automating the build and publication of a Python package (tarball/wheel) to a standalone JFrog Artifactory instance via a GitLab CI/CD pipeline.

To maintain security and separate environments, this workflow utilizes a dedicated **Technical Account** (Tech User) and targets a specific testing branch (`DEV-BUILD`) to prevent accidental modifications to production repositories.

## **2\. GitLab CI/CD Variables Configuration**

To authenticate securely with Artifactory without hardcoding credentials in the repository, we configured specific CI/CD variables at the GitLab project level.

1. Navigate to **Settings \> CI/CD \> Variables** in the GitLab repository.  
2. Add the following variables:  
   * **`technical_user`**: The username of the Artifactory technical account (e.g., `ap88353-tech-user`).  
   * **`technical_user_token`**: The authentication token/password for the technical account. **Note: This must be set to "Masked" to prevent it from appearing in pipeline logs.**

**\[Insert Screenshot Here: GitLab CI/CD Variables screen showing the masked token and user\]**

## **3\. Pipeline Configuration (`.gitlab-ci.yml`)**

We modified the `.gitlab-ci.yml` file to handle the specific behavior required for the `DEV-BUILD` branch.

### **3.1. Artifact Passing (`build-package` job)**

To ensure the packaged files are available to subsequent jobs, we defined the `artifacts` path. It is crucial that the folder name perfectly matches the output directory of the build script (in this case, `package/` without an 's').

YAML

```
build-package:
  stage: build-package
  # ... existing rules ...
  script:
    - echo ${APP_VERSION}
    - chmod +x ./scripts/build_tarball.sh
    - ./scripts/build_tarball.sh ${APP_NAME} ${APP_VERSION} ${PACKAGE_PATH}
  artifacts:
    paths:
      - "package/" # Ensures the generated package is passed to the next job
    expire_in: 1 day
```

**\[Insert Screenshot Here: Code snippet of the build-package job showing the artifacts path\]**

### **3.2. Sourcing Dependencies and Overriding Variables (`publish-package` job)**

In the publish stage, we override the standard credentials with our technical account variables exclusively for the `DEV-BUILD` branch. We also use the `dependencies` keyword to force the download of the artifacts from the build stage.

YAML

```
publish-package:
  stage: publish-package
  rules:
    - if: $CI_COMMIT_BRANCH == "DEV-BUILD"
      variables:
        ARTIFACTORY_USER: $technical_user
        ARTIFACTORY_PASSWORD: $technical_user_token
        REPOSITORY_SUFFIX: "SNAPSHOT" # Directs the upload to a test repository
    - !reference [.rule_iaas, rules]
    - !reference [.rule_kaas, rules]
  
  dependencies: 
    - "build-package" # Explicitly retrieves the 'package/' directory
  needs: ["pre-config", "build-package"]
  # ... script execution ...
```

**\[Insert Screenshot Here: Code snippet of the publish-package job showing the DEV-BUILD rule\]**

## **4\. Script Adjustments (`scripts/deploy_release.sh`)**

During deployment using the JFrog CLI, the technical account successfully uploaded the artifact but encountered a `403 Forbidden` error when attempting to publish "Build Info" metadata to the `artifactory-build-info` repository.

Since publishing build information is not necessary for this automated testing workflow, we bypassed this issue by commenting out the build-environment collection (`bce`) and build-publish (`bp`) commands.

Bash

```
# Upload the artifact to the specific local generic repository
jfrog rt u ${ARTIFACT} ap88353-generic-local/poc_registry/${ARTIFACT} --build-name=${APP_NAME} --build-number=${SCAN_BUILD_NBR} --module=${APP_NAME}

# The following lines are commented out to prevent 403 Forbidden errors, 
# as the tech user lacks permissions to write to the 'artifactory-build-info' repo:
# jfrog rt bce ${APP_NAME} ${SCAN_BUILD_NBR}
# jfrog rt bp ${APP_NAME} ${SCAN_BUILD_NBR}
```

**\[Insert Screenshot Here: The successful pipeline log showing the JFrog CLI upload bypassing the build info step\]**

