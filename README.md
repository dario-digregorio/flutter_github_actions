# Flutter App CI/CD with Fastlane and GitHub Actions
## Introduction

If you're looking to streamline your Flutter app's deployment to the AppStore and PlayStore, you're in the right spot. We'll use Fastlane and GitHub Actions to automate our workflow. By the end of this tutorial, you'll have a fully automated pipeline that builds, tests, versions and deploys your Flutter app to your preferred App Stores. This Guide will skip all the basic setup and configuration of Fastlane and GitHub Actions, and will focus on the Flutter and Workflow specific parts of the setup.

## Prerequisites

- A Flutter app (I'm assuming you've got this part covered 👀).
- Flavors and Environment Variables set up in your Flutter app. You can follow the [official Flutter guide](https://flutter.dev/docs/deployment/flavors) to set this up.
- You should be able to build a release version for Android and iOS. You can follow the [Android guide](https://flutter.dev/docs/deployment/android) [iOS guide](https://docs.flutter.dev/deployment/ios) to set this up.
- A GitHub account with your app's repo.
- Fastlane installed on your local machine. You can follow the [official Fastlane installation guide](https://docs.fastlane.tools/getting-started/ios/setup/).
- AppStore and PlayStore accounts ready for deployment with your apps project setup.
- Follow the initial [CD Flutter Guide](https://docs.flutter.dev/deployment/cd#local-setup) to setup the Fastlane folders in your Flutter project.

When you finished all the prerequisites, you should have following project structure with Fastlane folders in both iOS and Android folders:

```
- android
  - fastlane
    - Appfile
    - Fastfile
- ios
  - fastlane
    - Appfile
    - Fastfile
```
> Note: In this guide I will use the basic fastlane commands. It is highly recommended to use Fastlane with `bundle`. Read more about it in the Fastlane documentation

---
---

## Step 1: Setting Up The Fastfiles and Environment
Now we will setup the Fastfiles for both iOS and Android. We will define lanes for deploying the app to the AppStore and PlayStore.

### Android

1. Open `android/Gemfile` and add the flutter version plugin which we will need to extract the version from the pubspec.yaml file:

    ```ruby
    # ...
    source "https://rubygems.pkg.github.com/tianhaoz95" do
      gem "fastlane-plugin-flutter_version", "1.1.15"
    end
    ```

2. Open `android/fastlane/Appfile` and define the path to your service account JSON file and the package name of your app:

    ```ruby
    json_key_file("path/to/your/service-account.json")
    package_name("com.example.yourapp")
    ```
You created the Service Account JSON file in the [CD Flutter Guide](https://docs.flutter.dev/deployment/cd#local-setup). Make sure to exclude the file from your git repository.

3. Open `android/fastlane/Fastfile` and define the `deploy` lane (delete the existing content):

    ```ruby
    default_platform(:android)

    platform :android do
      desc "Deploy a new version to the Google Play"
      lane :deploy do
        upload_to_play_store(
          track: 'internal', # Can be 'internal', 'alpha', 'beta', 'production'
          skip_upload_metadata: true, # Skip uploading metadata
          skip_upload_images: true, # Skip uploading screenshots
          skip_upload_screenshots: true, # Skip uploading screenshots
          release_status: "completed", # Can be 'draft', 'completed', 'halted'
          aab: '../build/app/outputs/bundle/release/app-release.aab', # Path to your AAB file
          version_code: flutter_version()["version_code"], # From pubspec.yaml
          version_name: flutter_version()["version_name"] + flutter_version()["version_code"],
        )
      end
    end
    ```
    More information about the `upload_to_play_store` action can be found [here](https://docs.fastlane.tools/actions/upload_to_play_store/).
4. Run `bundle install` in the `android` directory to install the required gems.
5. Run `fastlane supply init` to initialize the PlayStore metadata. With this you can update the metadata like app description or screenshots without leaving your IDE. You can skip this step if you don't want to upload metadata. 
6. Build the app with `flutter build appbundle --release` and make sure the AAB file is located in the path you defined in the Fastfile and to use release signing keys. Follow the [Android guide](https://flutter.dev/docs/deployment/android) to set up the release signing.
7. Before deploying your first release you need to have at least one release already uploaded and published manually.
8. Run `fastlane deploy` to start the lane and deploy your app to the PlayStore. 

Depending on your setup, you might need to adjust the `upload_to_play_store` action to match your requirements.

### iOS

1. Open `ios/Gemfile` and add the flutter version plugin again:

    ```ruby
    # ...
    source "https://rubygems.pkg.github.com/tianhaoz95" do
      gem "fastlane-plugin-flutter_version", "1.1.15"
    end
    ```

2. Open `ios/fastlane/Appfile` and define app identifier, the Apple ID of your Apple Developer account and the Team ID:

    ```ruby
    app_identifier("com.example.yourapp")
    apple_id("test@your.email.com")
    itc_team_id("123456")
    ```


3. Open `ios/fastlane/Fastfile` and define the `deploy` lane (delete the existing content):

    ```ruby
    platform :ios do
      lane :deploy do
        pilot(
          skip_waiting_for_build_processing: true, # Skip waiting so we don't waist precious build time
          changelog: "This build was uploaded using fastlane",
          ipa: "../build/ios/ipa/flutter_github_actions.ipa" # Path to your IPA file
        )
      end
    end
    ```
    More information about [`pilot`](https://docs.fastlane.tools/actions/pilot/) and [`gym`](https://docs.fastlane.tools/actions/gym/) can be found in the Fastlane documentation.
4. Run `bundle install` in the `ios` directory to install the required gems.
5. Run `fastlane deliver init` to initialize the AppStore metadata. With this you can update the metadata like app description or screenshots without leaving your IDE. You can skip this step if you don't want to upload metadata.
6. Make sure you have already created an app in AppStore Connect.
7. Depending on the account you use you might need to further authenticate with an app-specific password. See the [Fastlane documentation](https://docs.fastlane.tools/getting-started/ios/authentication/) for more information.
8. Run `fastlane deploy` to start the lane and deploy your app to the AppStore. 

Depending on your setup, you might need to adjust the actions to match your requirements. Consider also to use the actions 

### Step 2: Setting Up GitHub Actions

Now that Fastlane is set up and you successfully run the lanes manually on you device, let's automate the deployment process with GitHub Actions.

> Note: To make the guide more simple we will use a self-hosted runner in this guide. You can also use the GitHub hosted runners. Make sure to adjust the paths and the Fastfile accordingly and to also add all necessary tools to the runner.

Now, let's automate these processes with GitHub Actions:

1. In your GitHub repo, create a new directory `.github/workflows`.
2. Create a new file in this directory, e.g. `deploy.yml`.
3. We need to create some secrets. Go to your repository settings and add the following secrets:
    - `STORE_PASSWORD`: The password for your keystore file.
    - `KEY_JKS`: The base64 encoded keystore file.
    - `SEC_JSON`: The base64 encoded service account JSON file.
    - `FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD`: The app-specific password for your Apple Developer account.
  
    To encode the files you can use the following command:
    ```bash
    cat path/to/your/file | openssl base64
    ```
    Copy the output and add it as a secret.
4. Add the following content to the `deploy.yml` file:
    ```yaml
    name: Build and Deploy
    on:
      workflow_dispatch:

    jobs:
      build_android:
        concurrency:
          group: ${{ github.workflow }}-${{ github.ref }}-android
          cancel-in-progress: true
        name: Build and Deploy Android
        runs-on: self-hosted
        env:
          STORE_PASSWORD: ${{ secrets.STORE_PASSWORD }} # The password for your keystore file
          KEY_JKS: ${{ secrets.KEY_JKS }} # The base64 encoded keystore file
          SEC_JSON: ${{ secrets.SEC_JSON }} # The base64 encoded service account JSON file

        steps:
          - uses: actions/checkout@v3
          - uses: actions/setup-java@v3.3.0
            with:
              distribution: "zulu"
              java-version: "17"
          - name: Create Key properties file
            run: |
                cat << EOF > "./android/key.properties"
                storePassword=${{ secrets.STORE_PASSWORD }}
                keyPassword=${{ secrets.STORE_PASSWORD }}
                keyAlias=upload
                storeFile=./key.jks
                EOF
          - name: Decode key file
            run: echo "${{ secrets.KEY_JKS }}" | openssl base64 -d -out ./android/app/key.jks
          - name: Decode sec json file
            run: echo "${{ secrets.SEC_JSON }}" | openssl base64 -d -out ./android/sec.json
          - uses: subosito/flutter-action@v2
          - run: flutter packages pub get
          - run: flutter build appbundle --release
          - name: Fastlane Action
            uses: maierj/fastlane-action@v2.3.0
            with:
              lane: deploy
              subdirectory: android

      build_ios:
        concurrency:
          group: ${{ github.workflow }}-${{ github.ref }}-ios
          cancel-in-progress: true
        name: Build and Deploy iOS
        runs-on: self-hosted
        env:
          FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD }}

        steps:
        - uses: actions/checkout@v3
        - uses: subosito/flutter-action@v2
        - run: flutter packages pub get
        - run: flutter build ipa --release
        - name: Deploy iOS Beta to TestFlight via Fastlane
          uses: maierj/fastlane-action@v2.3.0
          with:
            lane: deploy
            subdirectory: ios
    ```

    Let me break down what this workflow does. There are two jobs, `build_android` and `build_ios`. Each job builds and deploys the app to the PlayStore and AppStore respectively. The jobs are triggered manually by the `workflow_dispatch` event.
    To build the Android app we need to create the `key.properties` file and decode the keystore and service account JSON file. We then build the app bundle and run the Fastlane action with the `deploy` lane. The iOS job is simpler, we just build the IPA file and run the Fastlane action with the `deploy` lane.

5. Adjust the `flutter-version` as per your project's requirements.
6. Trigger the workflow manually by going to the Actions tab in your GitHub repository and selecting the `Build and Deploy` workflow. Click on the `Run workflow` button and select the branch you want to deploy.
7. Wait and watch the magic happen! 🎩✨

Whenever you want to deploy a new version of your app, simply edit the version in the `pubspec.yaml` and push your changes to the main branch and trigger the workflow manually. The workflow will build and deploy your app to the AppStore and PlayStore automatically.
### Conclusion

And that's it! You've now set up a CI/CD pipeline for your Flutter app using Fastlane and GitHub Actions. This setup will automatically build and deploy your app to the AppStore and PlayStore. Time to kick back, relax, and let automation handle the repetitive tasks. Happy coding! 🚀
