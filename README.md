![react-native-background-downloader banner](https://d1w2zhnqcy4l8f.cloudfront.net/content/falcon/production/projects/V5EEOX_fast/RNBD-190702083358.png)

[![npm version](https://badge.fury.io/js/@kesha-antonov%2Freact-native-background-downloader.svg)](https://badge.fury.io/js/@kesha-antonov%2Freact-native-background-downloader)

# @kesha-antonov/react-native-background-downloader
## *Note - This is fork of https://github.com/EkoLabs/react-native-background-downloader maintained by https://github.com/kesha-antonov * ##

A library for React-Native to help you download large files on iOS and Android both in the foreground and most importantly in the background.

### Why?
On iOS, if you want to download big files no matter the state of your app, wether it's in the background or terminated by the OS, you have to use a system API called `NSURLSession`.

This API handles your downloads separately from your app and only keeps it informed using delegates (Read: [Downloading Files in the Background](https://developer.apple.com/documentation/foundation/url_loading_system/downloading_files_in_the_background)).

On Android we are simulating this process with a wonderful library called [Fetch2](https://github.com/tonyofrancis/Fetch)

The real challenge of using this method is making sure the app's UI is always up-to-date with the downloads that are happening in another process because your app might startup from scratch while the downloads are still running.

`@kesha-antonov/react-native-background-downloader` gives you an easy API to both downloading large files and re-attaching to those downloads once your app launches again.

> **Please Note** - This library was created to better facilitate background downloading on iOS. If you're not aiming to to use the download-in-background functionality, there are better solutions like [RNFS.downloadFile()](https://github.com/itinance/react-native-fs#downloadfileoptions-downloadfileoptions--jobid-number-promise-promisedownloadresult-) which will results in a more stable download experience for your app.

## ToC

- [Usage](#usage)
- [API](#api)
- [Constants](#constants)

## Getting started

```Terminal
yarn add @kesha-antonov/react-native-background-downloader
```

or
```Terminal
npm i @kesha-antonov/react-native-background-downloader
```

Then:

```Terminal
cd ios && pod install
```

### Mostly automatic installation
Any React Native version **`>= 0.60`** supports autolinking so nothing should be done.

For anything **`< 0.60`** run the following link command

`$ react-native link @kesha-antonov/react-native-background-downloader`

### Manual installation

<details>

#### iOS

1. In XCode, in the project navigator, right click `Libraries` ➜ `Add Files to [your project's name]`
2. Go to `node_modules` ➜ `@kesha-antonov/react-native-background-downloader` and add `RNBackgroundDownloader.xcodeproj`
3. In XCode, in the project navigator, select your project. Add `libRNBackgroundDownloader.a` to your project's `Build Phases` ➜ `Link Binary With Libraries`
4. Run your project (`Cmd+R`)

#### Android

1. Open up `android/app/src/main/java/[...]/MainActivity.java`
  - Add `import com.eko.RNBackgroundDownloaderPackage;` to the imports at the top of the file
  - Add `new RNBackgroundDownloaderPackage()` to the list returned by the `getPackages()` method
2. Append the following lines to `android/settings.gradle`:
  	```
  	include ':react-native-background-downloader'
  	project(':react-native-background-downloader').projectDir = new File(rootProject.projectDir, 	'../node_modules/@kesha-antonov/react-native-background-downloader/android')
  	```
3. Insert the following lines inside the dependencies block in `android/app/build.gradle`:
  	```
      compile project(':react-native-background-downloader')
  	```
</details>

### iOS - Extra Mandatory Step
In your `AppDelegate.m` add the following code:
```objc
...
#import <RNBackgroundDownloader.h>

...

- (void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier completionHandler:(void (^)(void))completionHandler
{
  [RNBackgroundDownloader setCompletionHandlerWithIdentifier:identifier completionHandler:completionHandler];
}

...
```
Failing to add this code will result in canceled background downloads. If Xcode complains that RNBackgroundDownloader.h is missing, you might have forgotten to `pod install` first.

## Usage

### Downloading a file

```javascript
import { Platform } from 'react-native'
import { download, completeHandler } from '@kesha-antonov/react-native-background-downloader'

const jobId = 'file123'

let task = download({
	id: jobId,
	url: 'https://link-to-very.large/file.zip',
	destination: `${RNBackgroundDownloader.directories.documents}/file.zip`,
	metadata: {}
}).begin(({ expectedBytes, headers }) => {
	console.log(`Going to download ${expectedBytes} bytes!`)
}).progress(percent => {
	console.log(`Downloaded: ${percent * 100}%`)
}).done(() => {
	console.log('Download is done!')

  // PROCESS YOUR STUFF

  // FINISH DOWNLOAD JOB ON IOS
  if (Platform.OS === 'ios')
    completeHandler(jobId)
}).error(error => {
	console.log('Download canceled due to error: ', error);
})

// Pause the task
task.pause()

// Resume after pause
task.resume()

// Cancel the task
task.stop()
```

### Re-Attaching to background downloads

This is the main selling point of this library (but it's free!).

What happens to your downloads after the OS stopped your app? Well, they are still running, we just need to re-attach to them.

Add this code to app's init stage, and you'll never lose a download again!

```javascript
import RNBackgroundDownloader from '@kesha-antonov/react-native-background-downloader'

let lostTasks = await RNBackgroundDownloader.checkForExistingDownloads()
for (let task of lostTasks) {
	console.log(`Task ${task.id} was found!`)
	task.progress(percent => {
		console.log(`Downloaded: ${percent * 100}%`)
	}).done(() => {
		console.log('Download is done!')
	}).error(error => {
		console.log('Download canceled due to error: ', error)
	})
}
```

`task.id` is very important for re-attaching the download task with any UI component representing that task. This is why you need to make sure to give sensible IDs that you know what to do with, try to avoid using random IDs.

### Using custom headers
If you need to send custom headers with your download request, you can do in it 2 ways:

1) Globally using `RNBackgroundDownloader.setHeaders()`:
```javascript
RNBackgroundDownloader.setHeaders({
    Authorization: 'Bearer 2we$@$@Ddd223'
})
```
This way, all downloads with have the given headers.

2) Per download by passing a headers object in the options of `RNBackgroundDownloader.download()`:
```javascript
let task = RNBackgroundDownloader.download({
	id: 'file123',
	url: 'https://link-to-very.large/file.zip'
	destination: `${RNBackgroundDownloader.directories.documents}/file.zip`,
	headers: {
	    Authorization: 'Bearer 2we$@$@Ddd223'
	}
}).begin(({ expectedBytes, headers }) => {
	console.log(`Going to download ${expectedBytes} bytes!`)
}).progress(percent => {
	console.log(`Downloaded: ${percent * 100}%`)
}).done(() => {
	console.log('Download is done!')
}).error(error => {
	console.log('Download canceled due to error: ', error)
})
```
Headers given in the `download` function are **merged** with the ones given in `setHeaders`.

## API

### RNBackgroundDownloader

### `download(options)`

Download a file to destination

**options**

An object containing options properties

| Property      | Type                                             | Required | Platforms | Info                                                                                                                                                                             |
| ------------- | ------------------------------------------------ | :------: | :-------: | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`          | String                                           | ✅        | All       | A Unique ID to provide for this download. This ID will help to identify the download task when the app re-launches                                                               |
| `url`         | String                                           | ✅        | All       | URL to file you want to download                                                                                                                                                 |
| `destination` | String                                           | ✅        | All       | Where to copy the file to once the download is done                                                                                                                              |
| `metadata`    | Object                                           |           | All       | Data to be preserved on reboot.                                                                                                                              |
| `headers`      | Object                                           |          | All       | Costume headers to add to the download request. These are merged with the headers given in the `setHeaders` function
| `priority`    | [Priority (enum)](#priority-enum---android-only) |          | Android   | The priority of the download. On Android, downloading is limited to 4 simultaneous instances where further downloads are queued. Priority helps in deciding which download to pick next from the queue. **Default:** Priority.MEDIUM |
| `network`     | [Network (enum)](#network-enum---android-only)   |          | Android   | Give your the ability to limit the download to WIFI only. **Default:** Network.ALL                                                                                               |

**returns**

`DownloadTask` - The download task to control and monitor this download

### `checkForExistingDownloads()`

Checks for downloads that ran in background while you app was terminated. Recommended to run at the init stage of the app.

**returns**

`DownloadTask[]` - Array of tasks that were running in the background so you can re-attach callbacks to them

### `setHeaders(headers)`

Sets headers to use in all future downloads.

**headers** - Object

### DownloadTask

A class representing a download task created by `RNBackgroundDownloader.download`

### `Members`
| Name           | Type   | Info                                                                                                 |
| -------------- | ------ | ---------------------------------------------------------------------------------------------------- |
| `id`           | String | The id you gave the task when calling `RNBackgroundDownloader.download`                              |
| `metadata`     | Object | The metadata you gave the task when calling `RNBackgroundDownloader.download`                        |
| `percent`      | Number | The current percent of completion of the task between 0 and 1                                        |
| `bytesWritten` | Number | The number of bytes currently written by the task                                                    |
| `totalBytes`   | Number | The number bytes expected to be written by this task or more plainly, the file size being downloaded |

### `completeHandler(jobId)`

Finishes download job on iOS and informs OS that app can be closed in background if needed.
After finishing download in background you have some time to process your JS logic and finish the job.

### `ensureDownloadsAreRunning`

Pauses and resumes all downloads - this is fix for stuck downloads. Use it when your app loaded and is ready for handling downloads (all your logic loaded and ready to handle download callbacks).

Here's example of how you can use it:

1. When your app just loaded

Either stop all tasks:

```javascript
const tasks = await checkForExistingDownloads()
for (const task of tasks)
	task.stop()
```

Or re-attach them:

```javascript
const tasks = await checkForExistingDownloads()
for (const task of tasks) {
	task.pause()
	//
	// YOUR LOGIC OF RE-ATTACHING DOWLOADS TO YOUR STUFF
	// ...
	//
}
```

2. Prepare your app to handle downloads... (load your state etc.)

3. Add listener to handle when your app goes foreground (be sure to do it only after you stopped all tasks or re-attached them!)

```javascript

function handleAppStateChange (appState) {
	if (appState !== 'active')
		return

	ensureDownloadsAreRunning()
}

const appStateChangeListener = AppState.addEventListener('change', handleAppStateChange)
```

4. Call `ensureDownloadsAreRunning()` after all was setup.

### `Callback Methods`
Use these methods to stay updated on what's happening with the task.

All callback methods return the current instance of the `DownloadTask` for chaining.

| Function   | Callback Arguments                | Info                                                                                                                          |
| ---------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `begin`    | { expectedBytes, headers }        | Called when the first byte is received. 💡: this is good place to check if the device has enough storage space for this download |
| `progress` | percent, bytesWritten, totalBytes | Called at max every 1.5s so you can update your progress bar accordingly                                                      |
| `done`     |                                   | Called when the download is done, the file is at the destination you've set                                                   |
| `error`    | error                             | Called when the download stops due to an error                                                                                |

### `pause()`
Pauses the download

### `resume()`
Resumes a pause download

### `stop()`
Stops the download for good and removes the file that was written so far

## Constants

### directories

### `documents`

An absolute path to the app's documents directory. It is recommended that you use this path as the target of downloaded files.

### Priority (enum) - Android only

`Priority.HIGH`

`Priority.MEDIUM` - Default ✅

`Priority.LOW`

### Network (enum) - Android only

`Network.WIFI_ONLY`

`Network.ALL` - Default ✅

## Authors

Developed by [Elad Gil](https://github.com/ptelad) of [Eko](http://www.helloeko.com)

Maintained by [Kesha Antonov](https://github.com/kesha-antonov)

## License
Apache 2
