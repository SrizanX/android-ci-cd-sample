> [!IMPORTANT]  
> To run this project locally from **main** branch, you will need to add the following signing config credentials to your local.properties file
```properties
KEYSTORE_PASSWORD=123456
KEY_ALIAS=key0
KEY_PASSWORD=123456
```
# Let's Learn CI/CD in Android step by step
To successfully build our app, we require the following:
-   **Project source code**: The codebase for your Android application.
-   **A computer**: The machine where the build process will run.
-   **Gradle (via Wrapper)**: Gradle is included in the project as a wrapper, so no separate installation is required.
-   **Java compiler**: A Java Development Kit (JDK) compatible with the Gradle wrapper version used in the project.

From the project's root directory, we can build the desired APK variant by running the appropriate commands. The output APK will be generated in the `app/build/outputs/apk` directory.

| All variants         | Debug                      | Release                     |
|----------------------|----------------------------|-----------------------------|
| `./gradlew assemble` | ` ./gradlew assembleDebug` | `./gradlew assembleRelease` |

## Step 01. Build Signed APK
To build a signed APK, Android Studio provides a GUI for the task. However, in a CI/CD environment, where Android Studio is unavailable, we need to rely on the terminal for this process.

By running `./gradlew assemble`, an unsigned APK will be generated by default. To produce a signed APK, we need to configure the project to use the keystore (JKS) file. This requires editing the `app/build.gradle.kts` file.

Within the `android` block, we'll add a `signingConfigs {}` section to specify the necessary signing credentials (such as the keystore file, alias, and password). Once configured, we'll reference the created signing configuration in the `release` block under the `buildTypes` section. This ensures the APK is signed during the release build process.

```kotlin
android {
  signingConfigs {
    create("signingKey") {
	  storeFile = file("key_cicd_sample.jks")
	  storePassword = "123456"
	  keyAlias = "key0"
	  keyPassword = "123456"
	}
  }
  		
  release {
    //...
	signingConfig = signingConfigs.getByName("signingKey")
  }
}
```
Now if we run `./gradlew assemble` It will generate our signed apk.

[Step 01. Source code](https://github.com/SrizanX/android-ci-cd-sample/tree/01_add_signingConfig)

**Verify Signature (Optional)**
To verify the certificate used to sign the APK, we can use the `apksigner` tool from the terminal. This tool is located in `{your-sdk-folder}/build-tools/{version}`.
Run the following commands to check the signature of your APK:
```bash
apksigner verify --print-certs app/build/outputs/apk/debug/app-debug.apk
apksigner verify --print-certs app/build/outputs/apk/release/app-release.apk
```
In this example, the APKs were built locally. However, our next goal is to configure the project for APK signing within a CI/CD environment.

> [!CAUTION]
> We are currently providing the signing credentials directly within the `app/build.gradle.kts` file. If your project is public, this exposes sensitive information, which poses a security risk. Even in a private repository, embedding credentials in your code is considered poor practice. We will address this issue and implement a more secure solution in a later step.

## Step 02. Build APK automatically on push
There are several CI/CD solutions available, and for this project, we are using **GitHub Actions**. To implement this, we need to create a workflow file within our repository.

-   At the root of your project, create a `.github` directory.
-   Inside the `.github` directory, create another directory named `workflows`.
-   Within the `workflows` directory, create a `.yml` file and name it as desired.

Let's read our `.github/workflows/build.yml` workflow step by step:
**Run on push**
```yml
name: Build  
on:
  push:
    branches: [ "main" ]	
```
It will create a workflow which will run if you push on main `branch`. We have not created any jobs yet though.
**Create jobs and run on the Ubuntu operating system**
```yml
jobs:  
  build:  
    runs-on: ubuntu-latest  
    steps:  
      - name: Checkout sources  
        uses: actions/checkout@v4  
  
      - name: Setup JDK  
        uses: actions/setup-java@v4  
        with:  
          distribution: temurin  
          java-version: '17'
```
Our build process will run on a Linux desktop. We use the following GitHub Actions to automate the setup:

-   **Source Code Checkout**: We retrieve the project's source code using `actions/checkout@v4`, which is provided by GitHub.
-   **Java Setup**: To configure the appropriate Java version, we use `actions/setup-java@v4`, also provided by GitHub.

For further details on these actions, refer to their official documentation.

**Grant execute permission for gradle wrapper of our project**
```yml
      - name: Grant execute permission for gradlew  
        run: chmod +x ./gradlew
```
**Run assemble command with Gradle**
```yml
      - name: Build release variant with Gradle  
        run: ./gradlew assembleRelease
```
**Final workflow file**
```yml
name: Build  
on:  
  push:  
    branches: [ "main" ]  
jobs:  
  build:  
    runs-on: ubuntu-latest  
    steps:  
      - name: Checkout sources  
        uses: actions/checkout@v4  
  
      - name: Setup JDK  
        uses: actions/setup-java@v4  
        with:  
          distribution: temurin  
          java-version: '17'  
  
      - name: Grant execute permission for gradlew  
        run: chmod +x ./gradlew  
  
      - name: Build release variant with Gradle  
        run: ./gradlew assembleRelease
```
Once you commit and push your code to the `main` branch, GitHub Actions will automatically trigger the build process, and a signed release APK will be generated.

[Step 02. Source code](https://github.com/SrizanX/android-ci-cd-sample/tree/02_create_workflow)

## Step 03. Remove Signing Credentials from the Source Code
Storing sensitive credentials in source code is not secure. To enhance security, we will remove these credentials and instead retrieve them from system environment variables. We can access environment variables in our code using the `System.getenv("variable_key")` method.
### Defining Environment Variables in GitHub Actions
Since we're working on a virtual environment within GitHub Actions, we can define environment variables directly in our workflow `.yml` file:
```yml
env:  
  KEYSTORE_PASSWORD: 123456  
  KEY_ALIAS: key0 
  KEY_PASSWORD: 123456
```
[GitHub action variable docs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables)

Next, we will modify the `app/build.gradle.kts` file to fetch the `storePassword`, `keyAlias`, and `keyPassword` from the system environment variables:

```kotlin
signingConfigs {  
  create("signingKey"){  
	  storeFile = file("key_cicd_sample.jks")  
	  storePassword = System.getenv("KEYSTORE_PASSWORD")  
	  keyAlias = System.getenv("KEY_ALIAS")  
	  keyPassword = System.getenv("KEY_PASSWORD")  
  }  
}
```
However, since our `.yml` file is included in version control, the credentials would still be exposed in the repository. To mitigate this risk, we can use GitHub Secrets.
1.  Go to your repository's **Settings**.
2.  In the left panel, select **Secrets and variables** > **Actions**.
3.  Click on the **New repository secret** button and add your secrets, for example:

| Name              | Secret |
|-------------------|--------|
| KEYSTORE_PASSWORD | 123456 |
| KEY_ALIAS         | key0   |
| KEY_PASSWORD      | 123456 |

After adding your secrets, update your `.yml` file to reference these secrets securely:
```yml
env:  
  KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}  
  KEY_ALIAS: ${{ secrets.KEY_ALIAS }}  
  KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
```
This way, our sensitive information is securely stored in GitHub Secrets and not exposed in your codebase.

[Step 03. Source code](https://github.com/SrizanX/android-ci-cd-sample/tree/03_use_github_secrets)

## Step 04. Local Build System Issue
Our local build system is currently broken. Building the project through Android Studio or the local terminal fails because the required environment variables are not available in the local environment.

To fix this, we will use the `local.properties` file to store sensitive credential variables locally. This file is typically ignored by version control systems, but it's crucial to verify that it is indeed excluded from the repository. **Under no circumstances should this file be committed to Git.**

An example of the `local.properties` file for storing credentials:
```properties
KEYSTORE_PASSWORD=123456
KEY_ALIAS=key0
KEY_PASSWORD=123456
```
### Loading `local.properties` in `app/build.gradle.kts`
To load the properties from `local.properties`, use the following Kotlin code in your `build.gradle.kts`:
```kotlin
val localProperties = Properties()  
val localPropertiesFile = rootProject.file("local.properties")  
if (localPropertiesFile.exists()) {  
  localPropertiesFile.inputStream().use { localProperties.load(it) }  
}
```
### Updating the `signingConfigs` Block
Modify the `signingConfigs` block to use `local.properties` for local development builds and environment variables for CI/CD environments:
```kotlin
signingConfigs {
  create("signingKey") {  
    storeFile = file("key_cicd_sample.jks")
    if (localPropertiesFile.exists()) {  
        // Use local.properties for local builds  
        storePassword = localProperties["KEYSTORE_PASSWORD"] as String  
        keyAlias = localProperties["KEY_ALIAS"] as String  
        keyPassword = localProperties["KEY_PASSWORD"] as String  
    } else {  
        // Use environment variables for CI/CD  
        storePassword = System.getenv("KEYSTORE_PASSWORD")  
        keyAlias = System.getenv("KEY_ALIAS")
        keyPassword = System.getenv("KEY_PASSWORD")  
    }
  }
}
```
### Explanation
In this setup, we first check whether the `local.properties` file exists. If it does, this indicates that the build is running in a local development environment, and the credentials are loaded from the file. If not, the build is assumed to be in a CI/CD environment, and the credentials are retrieved from environment variables.

[Step 04. Source code](https://github.com/SrizanX/android-ci-cd-sample/tree/04_credentials_from_local_properties)