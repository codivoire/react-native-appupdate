[![npm version](https://badge.fury.io/js/@codivoire/react-native-appupdate.svg)](http://badge.fury.io/js/@codivoire/react-native-appupdate)
[![npm total downloads](https://img.shields.io/npm/dt/@codivoire/react-native-appupdate.svg)](https://img.shields.io/npm/dt/@codivoire/react-native-appupdate.svg)
[![npm monthly downloads](https://img.shields.io/npm/dm/@codivoire/react-native-appupdate.svg)](https://img.shields.io/npm/dm/@codivoire/react-native-appupdate.svg)
[![npm weekly downloads](https://img.shields.io/npm/dw/@codivoire/react-native-appupdate.svg)](https://img.shields.io/npm/dw/@codivoire/react-native-appupdate.svg)


# React Native Update APK

Easily check for new APKs and install them in React Native.

## Installation

```bash
npm install @codivoire/react-native-appupdate --save
```

Linking automatically with react-native link

```bash
react-native link @codivoire/react-native-appupdate
react-native link react-native-fs
```

## Manual steps for Android

1. **FileProviders:** Android API24+ requires the use of a FileProvider to share "content" (like
   downloaded APKs) with other applications (like the system installer, to install
   the APK update). So you must add a FileProvider entry to your [AndroidManifest.xml](example/android/app/src/main/AndroidManifest.xml),
   and it will reference a "[filepaths](example/android/app/src/main/res/xml/filepaths.xml)" XML file. Both are demonstrated in the example as linked here.

1. **Play Services** If you use Google Play Services, make sure to define 'googlePlayServicesVersion' as the correct version in [your main build.gradle](example/android/build.gradle) so you don't have crashes related to version mismatches. If you don't have 'com.google.android.gms:play-services-auth' as a dependency yet you will need to add it as a dependency in [your app build.gradle](example/android/app/build.gradle) as well - this is used to workaround SSL bugs for Android API16-20.

1. **Permissions** For Android 25+ you need to add REQUEST_INSTALL_PACKAGES to your [AndroidManifest.xml](example/android/app/src/main/AndroidManifest.xml)

**Please install and run the example to see how it works before opening issues**.
Then adapt it into your own app. Getting the versions right is tricky and setting up FileProviders is _very_ easy to do incorrectly, especially if using another module that defines one (like rn-fetch-blob)

## Example

```javascript
import React, { Component } from "react";
import { Alert, Button, SafeAreaView, ScrollView, StyleSheet, Text } from "react-native";
import AppUpdate from "@codivoire/react-native-appupdate";

type Props = {};
export default class App extends Component<Props> {
  constructor(props) {
    super(props);
    this.state = {
      // If you have something in state, you will be able to provide status to users
      downloadProgress: -1,
      allApps: [],
      allNonSystemApps: [],
    };

    updater = AppUpdate.update({

      // iOS must use App Store and this is the app ID. This is a sample: "All Birds of Ecuador" (¡Qué lindo!)
      iosAppId: "1104809018", 

      apkVersionUrl: "https://raw.githubusercontent.com/mikehardy/react-native-update-apk/master/example/test-version.json",

      //apkVersionOptions is optional, you should use it if you need to pass options to fetch request
      apkVersionOptions: {
        method:'GET',
        headers: {}
      },
      apkOptions: {
        headers: {}
      },
      fileProviderAuthority: "com.example.provider",

      // This callback is called if there is a new version but it is not a forceUpdate.
      needUpdateApp: needUpdate => {
        Alert.alert(
          "Update Available",
          "New version released, do you want to update? " +
            "(TESTING NOTE 1: stop your dev package server now - or the test package will try to load from it " +
            "instead of the included bundle leading to Javascript/Native incompatibilities." +
            "TESTING NOTE 2: the version is fixed at 1.0 so example test updates always work. " +
            "Compare the Last Update Times to verify it installed)",
          [
            { text: "Cancel", onPress: () => {} },
            { text: "Update", onPress: () => needUpdate(true) }
          ]
        );
      },
      
      // This will be called before the download/update where you defined forceUpdate: true in the version JSON
      forceUpdateApp: () => {
        console.log("forceUpdateApp callback called");
      },

      // Called if the current version appears to be the most recent available
      notNeedUpdateApp: () => {
        console.log("notNeedUpdateApp callback called");
      },

      // This is passed to react-native-fs as a callback
      downloadApkStart: () => {
        console.log("downloadApkStart callback called");
      },

      // Called with 0-99 for progress during the download
      downloadApkProgress: progress => {
        console.log(`downloadApkProgress callback called - ${progress}%...`);
        this.setState({ downloadProgress: progress });
      },
      
      // This is called prior to the update. If you throw it will abort the update
      downloadApkEnd: () => {
        console.log("downloadApkEnd callback called");
      },

      // This is called if the fetch of the version or the APK fails, so should be generic
      onError: err => {
        console.log("onError callback called", err);
        Alert.alert("There was an error", err.message);
      }
    });
  }

  async componentDidMount() {
    AppUpdate.patchSSLProvider()
      .then(ret => {
        // This means 
        console.log("SSL Provider Patch proceeded without error");
      })
      .catch(rej => {
        console.log("SSL Provider patch failed", rej);
        let message = "Old Android API, and SSL Provider could not be patched.";
        if (rej.message.includes("repairable")) {
          // In this particular case the user may even be able to fix it with a Google Play Services update
          message +=
            " This is repairable on this device though." +
            " You should send the users to the Play Store to update Play Services...";
          Alert.alert("Possible SSL Problem", message);
          UpdateAPK.patchSSLProvider(false, true); // This will ask Google Play Services to help the user repair
        } else {
          Alert.alert("Possible SSL Problem", message);
        }
      });

    AppUpdate.getApps().then(apps => {
      console.log("Installed Apps: ", JSON.stringify(apps));
      this.setState({ allApps: apps});
    }).catch(e => console.log("Unable to getApps?", e));

    AppUpdate.getNonSystemApps().then(apps => {
      console.log("Installed Non-System Apps: ", JSON.stringify(apps));
      this.setState({ allNonSystemApps: apps});
    }).catch(e => console.log("Unable to getNonSystemApps?", e));
  }

  _onCheckServerVersion = () => {
    console.log("checking for update");
    updater.checkUpdate();
  };

  render() {
    return (
      <SafeAreaView style={styles.container}>
        <Text style={styles.welcome}>rn-update-apk example</Text>
        <Text style={styles.instructions}>
          Installed Package Name: {UpdateAPK.getInstalledPackageName()}
        </Text>
        <Text style={styles.instructions}>
          Installed Version Code: {UpdateAPK.getInstalledVersionCode()}
        </Text>
        <Text style={styles.instructions}>
          Installed Version Name: {UpdateAPK.getInstalledVersionName()}
        </Text>
        <Text style={styles.instructions}>
          Installed First Install Time:
          {new Date(+UpdateAPK.getInstalledFirstInstallTime()).toUTCString()}
        </Text>
        <Text style={styles.instructions}>
          Installed Last Update Time:
          {new Date(+UpdateAPK.getInstalledLastUpdateTime()).toUTCString()}
        </Text>
        <Text style={styles.instructions}>
          Installed Package Installer:
          {UpdateAPK.getInstalledPackageInstaller()}
        </Text>
        <ScrollView style={{ flex: 1 }}>
          <Text style={styles.instructions}>
            Installed Apps: {JSON.stringify(this.state.allApps, null, '\t')}
          </Text>
          <Text style={styles.instructions}>
            Installed Non-System Apps: {JSON.stringify(this.state.allNonSystemApps, null, '\t')}
          </Text>
          <Text style={styles.instructions}>
            Installed Package Certificate SHA-256 Digest:
            { UpdateAPK.getInstalledSigningInfo() ? UpdateAPK.getInstalledSigningInfo()[0].thumbprint : "" }
          </Text>
          <Text style={styles.instructions}>
            { UpdateAPK.getInstalledSigningInfo() ? UpdateAPK.getInstalledSigningInfo()[0].toString : "" }
          </Text>
        </ScrollView>
        {this.state.downloadProgress != -1 && (
          <Text style={styles.instructions}>
            Download Progress: {this.state.downloadProgress}%
          </Text>
        )}
        <Button
          title="Check Server For Update"
          onPress={this._onCheckServerVersion}
        />
      </SafeAreaView>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: "center",
    alignItems: "center",
    backgroundColor: "#F5FCFF"
  },
  welcome: {
    fontSize: 20,
    textAlign: "center",
    margin: 10
  },
  instructions: {
    fontSize: 12,
    textAlign: "left",
    color: "#333333",
    marginBottom: 5
  }
});
```

## Usage

Please see [the example App.js](example/App.js) as it is very full featured and
has very thorough documentation about what each feature is for. You just need to check out the module from github, `cd example && npm install && npm start` then `react-native run-android` in another terminal with an emulator up to see everything in action.

## Changelog

See the [Changelog](CHANGELOG.md) on github

## Testing

This application has been tested on API16-API28 Android and will work for anything running API21+, plus any APIs between 16-20 that have Google Play Services. Specifically:

- API16 (Android 4.0) and up HTTP Updates + Emulators or Real Devices: works fine
- API21 (Android 5) and up HTTPS Updates + Emulators or Real Devices: works fine
- API16-API20 (Android 4.x) HTTPS Updates + Emulators: fails - platform SSL bug + no Google Play Services to patch it on these old emulators
- API16-API20 (Android 4.x) HTTPS Updates + Real Devices: works fine with Google Play Services to patch platform SSL bug

The only conditions where it won't work on real devices are for HTTPS updates to Android 4.x devices that do not have Google Play Services - a very very small percentage of the market at this point. Use HTTP if it is vital to reach those devices.

## Version JSON example

Note that you can host tests on dropbox.com using their "shared links", but if you do so
will need to put '?raw=1' at the end of the link they give you so you get the raw file contents
instead of a non-JSON XML document

```javascript
// version.json example
// Note you will need to verify SSL works for Android <5 as it has SSL Protocol bugs
// If it doesn't then you may be able to use Google Play Services to patch the SSL Provider, or just serve your updates over HTTP for Android <5
// https://stackoverflow.com/a/36892715
{
  "versionName":"1.0.0",
  "apkUrl":"https://github.com/NewApp.apk",
  "forceUpdate": false,
  "whatsNew": "<< what changes the app update will bring >>"
}
```

## Library Dependency

- react-native-fs
