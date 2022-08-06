## How to build and publish an Ionic app to App Store via Azure Pipeline

This is a concise guide on how to:
- Create an Ionic app from scratch
- Build ios package
- Publish the package to App Store

# Start a new project

Let's start a new project, code name `blackfish`.

First off, create a new folder `blackfish` and initialize a new Git repository with `git init`. This folder is where all your code stay.

## Create the Ionic App

In a real world app, you're most likely going to have to create a backend app at `server` or something. Going forward, let's assume that we already have everything about the server figured out.

1. Install the Ionic CLI

`npm i -g @ionic/cli`

2. Create the frontend app `client`.

`ionic start client`

This should start the Ionic wizard. Pick the options based on your own preferences. 

For the purpose of this guide, just pick `react`.

Once you're done, the CLI should create a blank project and install all dependencies for you already.

3. Change the app name and ID

Open [capacitor.config.ts](capacitor.config.ts) and change update it:

- Change `appId` to 'com.blackfish.client'
- Change `appName` to 'blackfish'

4. Add QA, Staging & Prod environment

In `client` folder, create a few `.env` files:

`.env`
```
REACT_APP_API_BASE_URL=https://localhost:8080
REACT_APP_ENV=local
```

`.env.qa`
```
REACT_APP_API_BASE_URL=https://localhost:8081
REACT_APP_ENV=qa
```

`.env.staging`
```
REACT_APP_API_BASE_URL=https://localhost:8082
REACT_APP_ENV=staging
```

`.env.prod`
```
REACT_APP_API_BASE_URL=https://localhost:8082
REACT_APP_ENV=prod
```

5. Add iOS platform

`ionic capacitor add ios`

5.1 Fix pod build

Because some error, we need to skip automatic code signing.

Open [podfile](client/ios/App/Podfile) and add this section right on top, just below `use_frameworks!` line. If there's already a similar section (starts with `post_install`), merge them together.

```
# workaround to get through xcode manual sign in on Azure Pipeline
# https://stackoverflow.com/questions/58975122/ionic-4-with-capacitor-xcodebuild-fails-with-manual-signing-process
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['EXPANDED_CODE_SIGN_IDENTITY'] = ""
      config.build_settings['CODE_SIGNING_REQUIRED'] = "NO"
      config.build_settings['CODE_SIGNING_ALLOWED'] = "NO"
    end
  end
end
```

6. If you already have the icons and splash screens prepared, you can go ahead and generate them

`cordova-res --skip-config --copy`

7. Test build

`ionic capacitor sync ios`

Since this is a blank project, it should not fail at this step. But if you're that unlucky. Uninstall your operation system, wipe your hard disk and try again.

## Build the project using Azure Pipeline

Since we can build the project locally without any error. Let's create the pipeline.

The first YML file only contains instructions to replicate the environment and execute the build command. With npm caching as cherry on top. You can omit if you want.

1. Create a template to build ionic app

See [build-ionic-app.yml](client/pipelines/build-ionic-app.yml)

Notice before we run build command, we rename the `.env.xxx` to `.env.production`. This is because `react-scripts` will use `production` environment by default when building.

Also, this template is not meant to be used by its own but will be called by other template. Some variables used in it will be set by the calling template.

2. Create a template to build ios package

See [build-ios-package.yml](client/pipelines/build-ios-package.yml)

3. Create a root template to build for QA environment

See [build-ios-qa.yml](client/pipelines/build-ios-qa.yml)

4. Create the pipeline

Once you have created and committed the 3 files above to your repository. Go to Azure Pipelines and create one pipeline, using the option to use existing template. Then point it to [build-ios-qa.yml](client/pipelines/build-ios-qa.yml)

## Collecting variables for the pipeline

Now you have the pipeline, but if you try to run it as is, you will have a bunch of errors.

Let's go through the list of variables you need, and how to obtain them.

1. Bundle ID & DevelopmentTeamId

- Go to [https://developer.apple.com/account/resources/identifiers/list]
- Create a new identifier `com.blackfish.client`
- Open the new identifier, write down the *Bundle ID* and *App ID Prefix (Team ID)*

2. Certificate

- Create a key file

```bash
openssl genrsa -out distribution.key 2048
```

- Create a *Certificate Signing Request* (CSR) file 

```bash
openssl req -new -sha256 -key distribution.key -out distribution.csr
```

- Go to [https://developer.apple.com/account/resources/certificates/list]
- Create a new Distribution Certificate with the CSR you just created
- Download the new certificate
- Convert it to p12 format

```bash
openssl x509 -inform DER -outform PEM -in distribution.cer -out distribution.pem
openssl pkcs12 -inkey distribution.key -in distribution.pem -export -out distribution.p12
```

- If you're running this on Windows, remember to add `winpty` prefix to the `openssl pkcs12` command to allow receiving inputs.

4. Provisioning Profile

- Go to [https://developer.apple.com/account/resources/profiles/list]
- Create a new Distribution Provisioning profile
- When asked, select the App ID and Certificate you have just created in earlier steps
- Download the profile

5. AppStore Upload Api Key

- Go to [https://appstoreconnect.apple.com/access/api]
- Write down the `Issuer ID`
- Create a new App Store Connect API key named `UploadToAppStoreKey`
- Write down the `Key ID`
- Download the key file, the name should be something like `AuthKey_xxxx.p8`
- Encode the entire text content of that file to `base64` format. You can use any tools
- For example: `certutil -encode AuthKey_xxxx.p8 out.txt`
- Copy the text content of `out.txt`

6. Apple ID

- Go to [https://appstoreconnect.apple.com/apps]
- Create a new App
- Open it
- Switch to `App Information` tab
- Write down the `Apple ID`

## Put the variables into Azure Library

1. Upload Secure files

- Upload `distribution.p12`
- Upload `Default_Provisioning_Profile.mobileprovision`

2. Create variable groups

- Create a group `ios-release-qa`
- Add variables:
   - AppleId
   - AppStoreUploadApiKeyEncoded
   - AppStoreUploadApiKeyId
   - AppStoreUploadApiKeyIssuerId
   - BundleId
   - CertificatePassword
   - DevelopmentTeamId

## Next? Let's do the same thing for Android

The process is the same, with some variations to the steps.

1. Add android platform

```bash
ionic capacitor add android
```

2. Create a template to build android package

See [build-android-package.yml](client/pipelines/build-android-package.yml)

3. Create a template to build QA environment

See [build-android-qa.yml](client/pipelines/build-android-qa.yml)

4. Create a keystore to sign the android package

```bash
echo y | keytool -genkeypair -dname "cn=Blackfish, ou=Software, o=Blackfish, c=US" -alias business -keypass averystrongpassword -keystore android.keystore -storepass evenstrongerpassword -validity 20000
```

Replace the name and password with your preferences.

5. Create Azure Library variable group `android-release-qa` and upload they keystore file to Secure Files.
