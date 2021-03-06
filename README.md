# About
docker-rnapk = Docker React Native Android application package building

This project aims to provide you with a command line tool based on Docker to build signed APK files for Android.

## Motivation
### Backstory
<details>
<summary>Show the Backstory</summary>

With "Creat React Native App" (CRNA) there exist an awesome tool for getting started with React Native. If you play with the thought to check out React Native you should definitly check out CRNA.

But "Creat React Native App" has limitations:
- for development only
- creats project which are in a kind of pre React Native state (in a context for production)
- it must be written in pure JavaScript
    - you can’t have any dependencies which rely on custom native code (which means you can't use native custome components)
- you have to installed the expo app on a mobile device (or simulator) to show/play around with your app
- devices (running the expo app) needs to be in the same network as the host (who runs a packeging server which streamlinse the JS package to the expo app)
    - sharing your app becomes way harder

At some point you want to distribute your app to coworkers, friends, or customers.
New developers who just startet with CRNA should also make a similar experience if it goes to production.

Here docker-rnapk joins the game.
</details>

### Goals
- create production ready _signed_ android apk files (this is the stuff you push to the app stores)
- works for CRNA (Creat React Native App) projects in the first place
- create reproducible results
- reduced time to setup for new developers (well you need to use and understand docker)
- you shouldn't need to install Android Studio (or NDK’s) as well
- be as flexible as possible (also support apps with native dependencies)
- enable APK signing
- provide a starting point for CI tools
- be as transparent as possible to understand the deployment/building process

## Get started

### 1. build the docker images
Before you can use this tool you have to create all needed docker images for that purpose.

> Be awere that creating these docker images automatically accepts all kinds of license agreements. It's by the way never a good idea to run any foreign scripts / creating docker images without checking whats happen in there.

_Note: Be awere that the combined size of the images takes more than 5GB of disk space._

Its recommended to execute the provided init.sh file (in the root directory of this repository) to ensure the naming of the images.

e.g. run: `bash init.sh`

Executing the init.sh script creates three images
- android-tools
- react-native-tools
- react-native-build

> the names of the images counts because react-native-build extends react-native-tools and this extends android-tools

Of course you can create each image manually by running (separately):
- `docker build -t android-tools ./android-tools`
- `docker build -t react-native-tools ./react-native-tools`
- `docker build -t react-native-build ./react-native-build`

### 2. Preparing the keystore
In order to create a signed apk file we need a keystore file. A keystore file basiclly contains one or more keys.
The contained key(s) are used to sign your apk files.

_Note: For an app you want to upload to the google play store you always have to use the exact same key._

Before you are going to create the keystore file think about proper names for the keystore file and the alias of the contained key you want to use to sign the apk file.
Also it's maybe a good idea to store the passwords you used to create the keystore file and the key on a secured place for you so you don't forget them.

_Note: the following is basiclly an extract from the [React Native documentation #generating-a-signing-key](https://github.com/facebook/react-native/blob/d0260b4f35f74a77d7f0d80d1e1b03233b92e978/docs/SignedAPKAndroid.md)_

<blockquote>
You can generate a private signing key using `keytool`. On Windows `keytool` must be run from `C:\Program Files\Java\jdkx.x.x_x\bin`.

    $ keytool -genkey -v -keystore my-release-key.keystore -alias my-key-alias -keyalg RSA -keysize 2048 -validity 10000

This command prompts you for passwords for the keystore and key, and to provide the Distinguished Name fields for your key. It then generates the keystore as a file called `my-release-key.keystore`.

The keystore contains a single key, valid for 10000 days. The alias is a name that you will use later when signing your app, so remember to take note of the alias.

_Note: Remember to keep your keystore file private and never commit it to version control._
</blockquote>

As the last note said, store the keystore file outside of your React Native Project.

For more information about apk signing [read the android developer document](https://developer.android.com/studio/publish/app-signing.html) about it.

This projects docker-rnapk is doing the actuall signing for you, so you don't have to worry about this technical step but you have to provide a keystore file for that purpose.

### 3. Create and configure a gradle.properties file
In order to tell the build script of the docker-rnapk container to use our keystore file and the containing key we have to expose the keystore filename, the right key alias and the password for the keystoore and the key.

We doing this through an gradle.properties configuration file.

`gradle.properties` file can look like:
```
    android.useDeprecatedNdk=true

    DOCKER_RNAPK_RELEASE_STORE_FILE=my-release-key.keystore
    DOCKER_RNAPK_RELEASE_KEY_ALIAS=my-key-alias
    DOCKER_RNAPK_RELEASE_STORE_PASSWORD=*****
    DOCKER_RNAPK_RELEASE_KEY_PASSWORD=*****
```

### 4. Prepare the signing-material directory
To propergate the the keystore file and the gradle.properties file to the docker-rnapk container we have to mount them. In order to do so you have to store them both in the same directory, in our example we call the directory "signing-material".

Then you have maybe this structure:
```
    /MyAppDirectory
       package.json
       app.josn
       ...
    /signing-material
        gradle.properties
        my-release-key.keystore
```

### 5. check / prepare your app project
In order to be deployable your app project directory needs:

- an index.js file (React Native >= 0.49.0) OR index.android.js file (React Native < 0.49.0)
    - index.android.js example [here](example/showcases/crna-based/some-crnapp/index.android.js)
    - index.js example [coming soon]()
- an app.json file
    - has to contain "name" and "displayName" key/value pairs [check out](https://github.com/facebook/react-native/blob/master/local-cli/eject/eject.js#L16) for the details
    - app.json example [here](example/showcases/crna-based/some-crnapp/app.json)

### 6. build the apk file
Before we finally build the apk file lets assume that your directory structure looks something like this:

```
    /MyAppDirectory
       package.json
       app.josn
       ...
    /signing-material
        gradle.properties
        my-release-key.keystore
```

Now we want to run the react-native-build image as container to build our apk file.
In order to do so we have at least to mount the app project and the signing-material into the docker container.

The final command should look something like that:

    docker run -it --rm --init -v $(pwd)/MyAppDirectory:/temp -v $(pwd)/signing-material:/apk-signing react-native-build

Now we have to wait until the container is done. Afterwards, bec. we used `--rm`, the container is automatically removed.

### 7. Find the apk file
The docker-rnapk project creats a "docker-rnapk-dist" folder in you project directory where you can find the signed apk file.

Precise location should be:
`/MyAppDirectory/docker-rnapk-dist/outputs/apk`

Done!

## example usage
`docker run -it --rm --init -v $(pwd):/temp -v $(pwd)/../apk-signing:/apk-signing react-native-build`

## container arguments

| Required | What                                | Argument/Example                                                  |
|----------|-------------------------------------|-------------------------------------------------------------------|
| yes      | project directory path              | -v $(pwd):/temp                                                   |
| yes      | apk-signing material directory path | -v /some/path:/apk-signing                                        |
| no       | gradle_deps volume                  | -v drnapk_gradle_deps:/gradle_deps                                |
| no       | node_modules volume                 | -v drnapk_node_modules:/temp-dist/node_modules                    |

_Note: Using the gradle_deps volume and node_modules volume will persit this folder / packages into docker volumes over multiple builds. This will improve the build speed._

## Testet with the following versions of React/React Native
| React             | React Native         |
|-------------------|----------------------|
| "16.0.0-alpha.12" | "0.45.1"             |

## Todo
- use docker volumn for handling node_modules
    - add a container env var to enabele (if it's wished) that the node_modules folder is cleared at the start
- integrate fastlane (optional) combined with https://github.com/butomo1989/docker-android
    - https://github.com/appfoundry/fastlane-android-example
- add a versioning script (so you can overwrite the current apk versioning)
- check this out for CI: https://medium.com/@tonespy/using-jenkins-pipeline-and-docker-for-android-continuous-integration-5fd39f8957a7
- check the app.json
- check for react-native version >= 0.49.0 for index.js vs index.ios.js and index.android.js
- check for currupt android/ios folder (not neccessary with rsync? or if exist enable custom build?)
- improve the keystore process, by providing a wrapper if you never had one bevore?
- provide example app for testing purpose and just for example?
- rename the used container to docker-rnapk instead of react-native-build
- do an external systemcheck for the docker container or something like that after each commit/deploy/new-version?
- add script arguments/flags e.q. --from-scratch to enable differentiation between crna vs post-crna
