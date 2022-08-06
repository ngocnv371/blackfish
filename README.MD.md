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
