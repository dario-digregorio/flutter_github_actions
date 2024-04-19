# Flutter App CI/CD with Fastlane and GitHub Actions
### Introduction

If you're looking to streamline your Flutter app's deployment to the AppStore and PlayStore, you're in the right spot. We'll use Fastlane for automation magic and GitHub Actions to automate our workflow. By the end of this tutorial, you'll have a fully automated pipeline that builds, tests, versions and deploys your Flutter app to your prefered App Stores. This Guide will skip all the basic setup and configuration of Fastlane and GitHub Actions, and will focus on the Flutter specific parts of the setup.

### Prerequisites

- A Flutter app (I'm assuming you've got this part covered 👀).
- Flavors and Environment Variables set up in your Flutter app. You can follow the [official Flutter guide](https://flutter.dev/docs/deployment/flavors) to set this up.
- App signing keys for both iOS and Android. You can follow the [Android guide](https://flutter.dev/docs/deployment/android) [iOS guide](https://docs.flutter.dev/deployment/ios) to set this up.
- A GitHub account with your app's repo already pushed.
- Fastlane installed on your local machine. You can follow the [official Fastlane installation guide](https://docs.fastlane.tools/getting-started/ios/setup/).
- AppStore and PlayStore accounts ready for deployment.


### Step 1: Setting Up Fastlane

First things first, let's get Fastlane set up in your project:

1. Navigate to your project directory in your terminal.
2. Run `fastlane init` and follow the on-screen instructions to set up Fastlane for iOS and Android.

For iOS, you'll need to provide your Apple ID and password to generate your app identifier and profiles. For Android, you'll need to have your signing keys ready.

### Step 2: Configure Fastlane

After initializing Fastlane, you'll find a `fastlane` folder in your project directory. Here's how to set it up for both platforms:

#### iOS

1. Open `fastlane/Fastfile` and define the `ios` lane:

    ```ruby
    desc "Deploy to the App Store"
    lane :ios do
      increment_build_number(xcodeproj: "Runner.xcodeproj")
      build_app(scheme: "Runner")
      upload_to_app_store(skip_metadata: true, skip_screenshots: true)
    end
    ```

2. Replace `"Runner.xcodeproj"` with your project's `.xcodeproj` file name if different.

#### Android

1. For Android, you'll need to edit the `fastlane/Appfile` to include your package name:

    ```ruby
    package_name("com.example.yourapp")
    ```

2. Then, define the `android` lane in `fastlane/Fastfile`:

    ```ruby
    desc "Deploy to the Play Store"
    lane :android do
      gradle(task: "assembleRelease")
      upload_to_play_store
    end
    ```

### Step 3: Setting Up GitHub Actions

Now, let's automate these processes with GitHub Actions:

1. In your GitHub repo, navigate to `Actions` > `New workflow`.
2. Choose "Set up a workflow yourself" and name it something like `flutter_ci_cd.yml`.
3. Replace the content with the following:

    ```yaml
    name: Flutter CI/CD

    on:
      push:
        branches:
          - main

    jobs:
      build-and-deploy:
        runs-on: ubuntu-latest
        steps:
        - uses: actions/checkout@v2
        - uses: actions/setup-java@v1
          with:
            java-version: '12.x'
        - uses: subosito/flutter-action@v1
          with:
            flutter-version: '2.x'
        - name: Install Fastlane
          run: |
            sudo gem install fastlane
        - name: Build and Deploy iOS
          if: github.ref == 'refs/heads/main'
          run: |
            cd ios
            fastlane ios
        - name: Build and Deploy Android
          if: github.ref == 'refs/heads/main'
          run: |
            cd android
            fastlane android
    ```

4. Adjust the `flutter-version` as per your project's requirements.

### Step 4: Commit and Push

Commit and push these changes to your GitHub repository. The `flutter_ci_cd.yml` GitHub Action will trigger on each push to the main branch, automating your build and deployment process.

### Conclusion

And that's it! You've now set up a CI/CD pipeline for your Flutter app using Fastlane and GitHub Actions. This setup will automatically build and deploy your app to the AppStore and PlayStore whenever you push changes to the main branch. Time to kick back, relax, and let automation handle the repetitive tasks. Happy coding! 🚀
