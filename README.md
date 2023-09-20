# BLE plugin for Apache Cordova with Nordic DFU

This plugin enables communication between a phone and Bluetooth Low Energy (BLE) peripherals. It is
a fork of excellent
[don/cordova-plugin-ble-central](https://github.com/don/cordova-plugin-ble-central) plugin enriched
with Nordic Semiconductors [Android](https://github.com/NordicSemiconductor/Android-DFU-Library) and
[iOS](https://github.com/NordicSemiconductor/IOS-Pods-DFU-Library) DFU libraries.

**For the main documentation, please visit the [base plugin GitHub
page](https://github.com/don/cordova-plugin-ble-central).**
This page covers only additional installation requirements and extended API.

---
-   Scan for peripherals
-   Connect to a peripheral
-   Read the value of a characteristic
-   Write new value to a characteristic
-   Get notified when characteristic's value changes
-   Transfer data via L2CAP channels

# Requirements

For using this plugin on iOS, there are some additional requirements:

- `cordova` version >= 6.4
- `cordova-ios` version >=4.3.0
- [CocoaPods](https://guides.cocoapods.org/using/getting-started.html#installation)

# Installing

    $ cordova plugin add https://github.com/flemmingdjensen/cordova-plugin-ble-central.git

# Extended API

### PhoneGap Build

-   `accessBackgroundLocation`: [**Android**] Tells the plugin to request the `ACCESS_BACKGROUND_LOCATION` permission on Android 10 & Android 11 in order for scanning to function when the app is not visible.

### iOS

After installation, the following additions should be made to the app's `Info.plist`

-   Set [NSBluetoothAlwaysUsageDescription](https://developer.apple.com/documentation/bundleresources/information_property_list/nsbluetoothalwaysusagedescription?language=objc) to a descriptive text, to be shown to the user on first access to the Bluetooth adapter. **If this is not defined the app will crash**.
-   _(Optional)_ Add `bluetooth-central` to [UIBackgroundModes](https://developer.apple.com/documentation/bundleresources/information_property_list/uibackgroundmodes?language=objc) to enable background receipt of scan information and BLE notifications

## Cordova

`$ cordova plugin add cordova-plugin-ble-central --variable BLUETOOTH_USAGE_DESCRIPTION="Your description here" --variable IOS_INIT_ON_LOAD=true|false --variable BLUETOOTH_RESTORE_STATE=true|false --variable ACCESS_BACKGROUND_LOCATION=true|false`

It's recommended to always use the latest cordova and cordova platform packages in order to ensure correct function. This plugin generally best supports the following platforms and version ranges:

| cordova | cordova-ios | cordova-android | cordova-browser |
| ------- | ----------- | --------------- | --------------- |
| 10+     | 6.2.0+      | 10.0+           | not tested      |

All variables can be modified after installation by updating the values in `package.json`.

-   `BLUETOOTH_USAGE_DESCRIPTION`: [**iOS**] defines the values for [NSBluetoothAlwaysUsageDescription](https://developer.apple.com/documentation/bundleresources/information_property_list/nsbluetoothalwaysusagedescription?language=objc).

-   `IOS_INIT_ON_LOAD`: [**iOS**] Prevents the Bluetooth plugin from being initialised until first access to the `ble` window object. This allows an application to warn the user before the Bluetooth access permission is requested.

-   `BLUETOOTH_RESTORE_STATE`: [**iOS**] Enable Bluetooth state restoration, allowing an app to be woken up to receive scan results and peripheral notifications. This is needed for background scanning support. See [iOS restore state](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html#//apple_ref/doc/uid/TP40013257-CH7-SW13). For more information about background operation with this plugin, see [Background Scanning and Notifications on iOS](#background-scanning-and-notifications-on-ios).

-   `ACCESS_BACKGROUND_LOCATION`: [**Android**] Tells the plugin to request the ACCESS_BACKGROUND_LOCATION permission on Android 10 & Android 11 in order for scanning to function when the app is not visible.

## Android permission conflicts

If you are having Android permissions conflicts with other plugins, try using the `slim` variant of the plugin instead with `cordova plugin add cordova-plugin-ble-central@slim`. This variant excludes all Android permissions, leaving it to the developer to ensure the right entries are added manually to the `AndroidManifest.xml` (see below for an example).

To include the default set of permissions the plugin installs on Android SDK v31+, add the following snippet in your `config.xml` file, in the `<platform name="android">` section:

- [all methods from the original plugin](https://github.com/don/cordova-plugin-ble-central#methods)
- [`ble.upgradeFirmware`](#upgradeFirmware)

## upgradeFirmware

Upgrade a peripheral firmware.

```javascript
ble.upgradeFirmware(device_id, uri, progress, failure);
```

### Description

Function `upgradeFirmware` upgrades peripheral firmware using the Nordic Semiconductors'
[proprietary DFU
protocol](https://devzone.nordicsemi.com/documentation/nrf51/6.0.0/s110/html/a00062.html) (hence
only Nordic nRF5x series devices can be upgraded). It uses the official DFU libraries for each
platform and wraps them for use with Apache Cordova. Currently only supported firmware format is a
[ZIP file](https://devzone.nordicsemi.com/blogs/715/creating-zip-package-for-dfu/) prepared using
Nordic CLI utilities.

The function presumes a connected BLE peripheral. A progress callback is called multiple times with
upgrade status info, which is a JSON object of the following format:

```javascript
{
    "status": "--machineFriendlyString--"
}
```

A complete list of possible status strings is:

- `deviceConnecting`
- `deviceConnected`
- `enablingDfuMode`
- `dfuProcessStarting`
- `dfuProcessStarted`
- `firmwareUploading`
- `progressChanged` - extended status info
- `firmwareValidating`
- `dfuCompleted`
- `deviceDisconnecting`
- `deviceDisconnected` - the last callback on successful upgrade
- `dfuAborted` - the last callback on user abort

The list is only approximately ordered. *Not all statuses all presented on both platforms.*
If `status` is `progressChanged`, the object is extended by a `progress` key like so:

```javascript
{
    "status": "progressChanged",
    "progress": {
        "percent": 12,
        "speed": 2505.912325285,
        "avgSpeed": 1801.8598291,
        "currentPart": 1,
        partsTotal: 1
    }
}
```

In a case of error, the JSON object passed to failure callback has following structure:

```javascript
{
    "errorMessage": "Hopefully human readable error message"
}
```

Please note, that the device will disconnect (possibly multiple times) during the upgrade, so the
[ble.connect](https://github.com/don/cordova-plugin-ble-central#connect) error callback will
trigger. This is intentional.

### Parameters

- __device_id__: UUID or MAC address of the peripheral
- __uri__: URI of a firmware ZIP file on the **local filesystem**
  (see [cordova-plugin-file](https://github.com/apache/cordova-plugin-file))
- __progress__: Progress callback function that is invoked multiple times with upgrade status info
- __failure__: Error callback function, invoked when an error occurs

### Quick Example

```javascript
// presume connected device

var device_id = "BD922605-1B07-4D55-8D09-B66653E51BBA"
var uri = "file:///var/mobile/Applications/12312-1231-1231-123312-123123/Documents/firmware.zip";

ble.upgradeFirmware(device_id, uri, console.log, console.error);
```
