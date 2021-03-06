# VimeoUpload

This library is under active development. We're shooting for a v1.0 release soon. All comments, questions, pull requests, and issues are welcome. See below for details.

## Contents
* [Design Considerations](#design-considerations)
      * [Old Upload, New Upload 👴👶](#old-upload-new-upload)
      * [Constraints](#constraints)
      * [Goals](#goals)
      * [Anatomy](#anatomy)
          * [NSURLSession](#nsurlsession)
          * [AFNetworking](#afnetworking)
          * [VimeoUpload](#vimeoupload)
* [Getting Started](#getting-started)
      * [Prerequisites](#prerequisites)
      * [Example Projects](#example-projects)
      * [CocoaPods](#cocoapods)
      * [Submodule](#submodule)
      * [Initialization](#initialization)
* [Uploading Videos](#uploading-videos)
     * [Starting an Upload](#starting-an-upload)
          * [Obtaining a File URL For a PHAsset](#obtaining-a-file-url-for-a-phasset)
          * [Obtaining a File URL For an ALAsset](#obtaining-a-file-url-for-an-alasset)
          * [Obtaining a File URL For an Asset That You Manage](#obtaining-a-file-url-for-an-asset-that-you-manage)
          * [Start Your Upload](#start-your-upload)
     * [Inspecting Upload State and Progress](#inspecting-upload-state-and-progress)
     * [Canceling an Upload](#canceling-an-upload)
* [Custom Workflows 🍪🎉](#custom-workflows)
* [Want to Contribute?](#want-to-contribute)
* [Found an Issue?](#found-an-issue)
* [Questions](#questions)
* [License](#license)

## Design Considerations

### Old Upload, New Upload

The current (public) server-side Vimeo upload API is comprised of 4 separate requests that must be made in sequence. This is more complex than we'd like it to be, and this complexity is not ideal for native mobile clients. More requests means more failure points. More requests means a process that's challenging to communicate to the developer and in turn to the user. The 4 requests are: 

1. Create a video object 
2. Upload the video file
3. Activate the video object 
4. Optionally set the video object's metadata (e.g. title, description, privacy, etc.)

We affectionately refer to this 4-step flow as Old Upload. 

A simplified flow that eliminates steps 3 and 4 above is in private beta right now. It's being used in the current [Vimeo iOS](https://itunes.apple.com/app/id425194759) and [Vimeo Android](https://play.google.com/store/apps/details?id=com.vimeo.android.videoapp) apps and it's slated to be made available to the public later this year.

We affectionately refer to this 2-step flow as New Upload.

VimeoUpload is designed to accommodate a variety of [background task workflows](#custom-workflows) including Old Upload and New Upload. The library currently contains support for both. However, New Upoad classes are currently marked as "private" and will not work for the general public until they are released from private beta.

The VimeoUpload APIs for New and Old Upload are very similar. The Old Upload API is documented below. Old upload will be deprecated as soon as possible and the README will be updated to reflec the New Upload API at that time.

### Constraints

* **iOS 7 Support Required** 

 Because the Vimeo iOS app supports iOS7, and because a bunch of other apps out there do too, we must support both the [ALAsset](https://developer.apple.com/library/ios/documentation/AssetsLibrary/Reference/ALAssetsLibrary_Class/) and [PHAsset](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHAsset_Class/index.html) APIs.

* **No Direct Access to PHAsset or ALAsset Source Files** 

 Because of how the Apple APIs are designed, we cannot upload directly from [ALAsset](https://developer.apple.com/library/ios/documentation/AssetsLibrary/Reference/ALAssetsLibrary_Class/) and [PHAsset](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHAsset_Class/index.html) source files. So we must export a copy of the asset using an [AVAssetExportSession](https://developer.apple.com/library/prerelease/ios/documentation/AVFoundation/Reference/AVAssetExportSession_Class/index.html) before starting the upload process. Asset export must happen when the app is in the foreground.

* **iCloud Photos** 

 If a [PHAsset](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHAsset_Class/index.html) is in iCloud and not resident on device we need to download it to the device before asset export. Download must happen when the app is in the foreground.

* **Background Sessions** 

 Because an upload can take a significant amount of time, we must design for the user potentially backgrounding the application at any point in the process. Therefore all requests must be handled by an [NSURLSession](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSession_class/) configured with a background [NSURLSessionConfiguration](https://developer.apple.com/library/prerelease/ios/documentation/Foundation/Reference/NSURLSessionConfiguration_class/index.html). This means we must rely exclusively on the [NSURLSessionDelegate](https://developer.apple.com/library/prerelease/ios/documentation/Foundation/Reference/NSURLSessionDelegate_protocol/index.html), [NSURLSessionTaskDelegate](https://developer.apple.com/library/prerelease/ios/documentation/Foundation/Reference/NSURLSessionTaskDelegate_protocol/index.html#//apple_ref/occ/intf/NSURLSessionTaskDelegate), and [NSURLSessionDownloadDelegate](https://developer.apple.com/library/prerelease/ios/documentation/Foundation/Reference/NSURLSessionDownloadDelegate_protocol/index.html#//apple_ref/occ/intf/NSURLSessionDownloadDelegate) protocols. We cannot rely on an [NSURLSessionTask](https://developer.apple.com/library/prerelease/ios/documentation/Foundation/Reference/NSURLSessionTask_class/index.html) subclasses' completion blocks.
 
* **Resumable Uploads**

 The [NSURLSession](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSession_class/) API does not support resuming an interrupted background upload from an offset. The initial release of this library will use these APIs exclusively and therefore will also not support resuming an upload from an offset. 

* **Fault Tolerance** 

 The app process could be backgrounded, terminated, or crash at any time. This means that we need to persist the state of the upload system to disk so that it can be reconstructed as needed. Crashes should not adversely impact uploads or the upload system.

* **Concurrent File Uploads** 

 We want to be able to conduct concurrent uploads.

* **State Introspection**

 We want to be able to communicate upload progress and state to the user in a variety of locations.  Including communicating state for uploads initiated on other devices and other platforms.

* **Reachability Awareness** 

 The app could lose connectivity at any time. We want to be able to pause and resume the upload system when we lose connectivity. And we want to be able to allow the user to restrict uploads to wifi.

* **Upload Quotas** 

 We need to be able to communicate information to users about their [upload quota](https://vimeo.com/help/faq/uploading-to-vimeo/uploading-basics). 

* **iOS SDK Quirks** 

 NSURLSessionTask's `suspend` method doesn't quite do what we want it to `TODO: Provide details`
 
 NSURLRequest's and NSURLSessionConfiguration's `allowsCellularAccess` properties don't quite do what we want them to `TODO: provide details`
 
 AFURLSessionManager's `tasks` property behaves differently on iOS7 vs. iOS8+ `TODO: provide details`
 
 And more...

### Goals

1. A simplified server-side upload API

1. An upload system that addresses each [constraint](#constraints) listed above

1. Clear and concise upload system initialization, management, and introspection

1. An upload system that accommodates as many UX futures as possible

### Anatomy

#### NSURLSession

TODO

#### AFNetworking

TODO

#### VimeoUpload

TODO

## Getting Started

### Prerequisites

1. Ensure that you've verified your Vimeo account. When you create an account, you'll receive an email asking that you verify your account. Until you verify your account you will not be able to upload videos using the API. 
2. Ensure you have been granted permission to use the "upload" scope. This permission must explicitly be granted by Vimeo API admins. You can request this permission on your app page under "Request upload access". Visit [developer.vimeo.com](https://developer.vimeo.com/).
3. Ensure that the OAuth token that you're using to make your requests has the "upload" scope included.

### Example Projects

There's an example project for New Upload and one for Old Upload. In order to run them you'll have to drop a valid OAuth token into the example project's `VimeoUpload` subclass' `init` method where it says `"YOUR_OAUTH_TOKEN"`. You can obtain an OAuth token by visiting [developer.vimeo.com](https://developer.vimeo.com/apps) and creating a new "app" and associated OAuth token.

### CocoaPods

TODO

### Submodule

TODO

### Initialization

Create an instance of `VimeoUpload`, or modify `VimeoUpload` to act as a singleton: 

```Swift
    let backgroundSessionIdentifier = "YOUR_BACKGROUND_SESSION_ID"
    let authToken = "YOUR_OAUTH_TOKEN"
    
    let vimeoUpload = VimeoUpload<OldUploadDescriptor>(backgroundSessionIdentifier: backgroundSessionIdentifier, authToken: authToken)
```

If your OAuth token can change during the course of a session, use the constructor whose second argument is an `authTokenBlock`:

```Swift
    let backgroundSessionIdentifier = "YOUR_BACKGROUND_SESSION_ID"
    var authToken = "YOUR_OAUTH_TOKEN"

    let vimeoUpload = VimeoUpload<OldUploadDescriptor>(backgroundSessionIdentifier: backgroundSessionIdentifier, authTokenBlock: { () -> String? in
        return authToken 
    })
```

You can obtain an OAuth token by using the authentication methods provided by [VIMNetworking](https://github.com/vimeo/VIMNetworking) or by visiting [developer.vimeo.com](https://developer.vimeo.com/apps) and creating a new "app" and associated OAuth token.

## Uploading Videos

### Starting an Upload

In order to start an upload, you need a file URL pointing to the video file on disk that you would like to upload.

The steps required to obtain the file URL will vary depending on whether you are uploading a [PHAsset](#obtaining-a-file-url-for-a-phasset), an [ALAsset](#obtaining-a-file-url-for-an-alasset), or an [asset that you manage](#obtaining-a-file-url-for-an-asset-that-you-manage) outside of the device Photos environment. Once you have a valid file URL, you will use it to [start your upload](#start-your-upload).

Unfortunately, because of how Apple's [PHAsset](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHAsset_Class/index.html) and [ALAsset](https://developer.apple.com/library/ios/documentation/AssetsLibrary/Reference/ALAssetsLibrary_Class/) APIs are designed uploading directly from a [PHAsset](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHAsset_Class/index.html) or [ALAsset](https://developer.apple.com/library/ios/documentation/AssetsLibrary/Reference/ALAssetsLibrary_Class/) resource URL is not possible. In order to upload [PHAsset](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHAsset_Class/index.html)s and [ALAsset](https://developer.apple.com/library/ios/documentation/AssetsLibrary/Reference/ALAssetsLibrary_Class/)s you will need to first create a copy of the asset itself and upload from that copy. See below for instructions on how to do this. 

#### Obtaining a File URL For a PHAsset

Use VimeoUpload's `PHAssetExportSessionOperation` to request an instance of [AVAssetExportSession](https://developer.apple.com/library/prerelease/ios/documentation/AVFoundation/Reference/AVAssetExportSession_Class/index.html) configured for the [PHAsset](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHAsset_Class/index.html) that you intend to upload. If the [PHAsset](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHAsset_Class/index.html) is in iCloud (i.e. not resident on the device) this will download the [PHAsset](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHAsset_Class/index.html) from iCloud. 

```Swift 
    let phAsset = ... // The PHAsset you intend to upload
    let operation = PHAssetExportSessionOperation(phAsset: phAsset)
    
    // Optionally set a progress block
    operation.progressBlock = { (progress: Double) -> Void in
        // Do something with progress
    }
    
    operation.completionBlock = {
        guard operation.cancelled == false else
        {
            return
        }

        if let error = operation.error
        {
            // Do something with the error
        }
        else if let exportSession = operation.result
        {
            // Use the export session to export a copy of the asset (see below)
        }
        else
        {
            assertionFailure("error and exportSession are mutually exclusive. This should never happen.")
        }
    }

    operation.start()
```

Next, use VimeoUpload's `ExportOperation` to export a copy of the [PHAsset](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHAsset_Class/index.html). You can then use the resulting `url` to [start your upload](#start-your-upload).

```Swift
    let exportSession = ... // The export session you just generated (see above)
    let operation = ExportOperation(exportSession: exportSession)
    
    // Optionally set a progress block
    operation.progressBlock = { (progress: Double) -> Void in
        // Do something with progress
    }
    
    operation.completionBlock = {
        guard operation.cancelled == false else
        {
            return
        }

        if let error = operation.error
        {
            // Do something with the error
        }
        else if let url = operation.outputURL
        {
            // Use the url to start your upload (see below)
        }
        else
        {
            assertionFailure("error and outputURL are mutually exclusive, this should never happen.")
        }
    }
    
    operation.start()
```

#### Obtaining a File URL For an ALAsset

Use VimeoUpload's `ExportOperation` to export a copy of the [ALAsset](https://developer.apple.com/library/ios/documentation/AssetsLibrary/Reference/ALAssetsLibrary_Class/) you intend to upload. You can then use the resulting `url` to [start your upload](#start-your-upload).

```Swift
    let alAsset = ... // The ALAsset you intend to upload
    let url = alAsset.defaultRepresentation().url() // For example
    let avAsset = AVURLAsset(URL: url)
    let operation = ExportOperation(asset: avAsset)
    
    // Optionally set a progress block
    operation.progressBlock = { (progress: Double) -> Void in
        // Do something with progress
    }
    
    operation.completionBlock = {
        guard operation.cancelled == false else
        {
            return
        }

        if let error = operation.error
        {
            // Do something with the error
        }
        else if let url = operation.outputURL
        {
            // Use the url to start your upload (see below)
        }
        else
        {
            assertionFailure("error and outputURL are mutually exclusive, this should never happen.")
        }
    }
    
    operation.start()
```

#### Obtaining a File URL For an Asset That You Manage

This is quite a bit simpler: 

```Swift
    let path = "PATH_TO_VIDEO_FILE_ON_DISK"
    let url = NSURL.fileURLWithPath(path)
```

#### Start Your Upload

Use the `url` to start your upload: 

```Swift
    let vimeoUpload = ... // Your instance of VimeoUpload (see above)
    let url = ... // Your url (see above)

    let descriptor = OldUploadDescriptor(url: url)
    vimeoUpload.uploadVideo(descriptor: descriptor)
```

You can also pass in a VideoSettings object if you'd like. This will set your video's metadata after the upload completes: 

```Swift
    let vimeoUpload = ... // Your instance of VimeoUpload (see above)
    let url = ... // Your url (see above)
    
    let title = "Untitled"
    let description = "A really cool video"
    let privacy = "nobody"
    let videoSettings = VideoSettings(title: title, description: description, privacy: privacy, users: nil, password: nil)
    
    let descriptor = OldUploadDescriptor(url: url, videoSettings: videoSettings)
    vimeoUpload.uploadVideo(descriptor: descriptor)
```

You can use the descriptor you create to [inspect state and progress](#inspecting-upload-state-and-progress), or to [cancel the upload](#canceling-an-upload).

### Inspecting Upload State and Progress

You can examine upload state and progress by inspecting the `stateObservable` and `progressObservable` properties of an `OldUploadDescriptor`. 

You can obtain a reference to a specific `OldUploadDescriptor` by holding onto the `OldUploadDescriptor` that you used to initiate the upload, or by asking `VimeoUpload` for a specific `OldUploadDescriptor` like so:

```Swift
    let identifier = ... // The identifier you set on the descriptor when you created it (see above)
    let vimeoUpload = ... // Your instance of VimeoUpload (see above)
    let descriptor = vimeoUpload.descriptorForIdentifier(identifier: identifier)
```

You can also ask `VimeoUpload` for an `OldUploadDescriptor` that passes a test that you construct. You can construct any test you'd like. In the case where we want a descriptor with a certain identifier, the convenience method `descriptorForIdentifier` leverages `descriptorPassingTest` under the hood:

```Swift
    let phAsset = ... // The PHAsset whose upload you'd like to inspect
    let vimeoUpload = ... // Your instance of VimeoUpload (see above)
    
    let descriptor = vimeoUpload.descriptorManager.descriptorPassingTest({ (descriptor) -> Bool in
        return descriptor.identifier == phAsset.localIdentifier   
    })
```

Once you have a reference to the `OldUploadDescriptor` you're interested in you can inspect its state directly:

```Swift
    print(descriptor.state)
    print(descriptor.error?.localizedDescription)
```

Or use KVO to observe changes to its state and progress: 

```Swift
    private static let ProgressKeyPath = "progressObservable"
    private static let StateKeyPath = "stateObservable"
    private var progressKVOContext = UInt8()
    private var stateKVOContext = UInt8()

    ...
    
    descriptor.addObserver(self, forKeyPath: self.dynamicType.StateKeyPath, options: .New, context: &self.stateKVOContext)
    descriptor.addObserver(self, forKeyPath: self.dynamicType.ProgressKeyPath, options: .New, context: &self.progressKVOContext)
    
    ...
    
    descriptor.removeObserver(self, forKeyPath: self.dynamicType.StateKeyPath, context: &self.stateKVOContext)
    descriptor.removeObserver(self, forKeyPath: self.dynamicType.ProgressKeyPath, context: &self.progressKVOContext)

    ...
    
    // MARK: KVO
    
    override func observeValueForKeyPath(keyPath: String?, ofObject object: AnyObject?, change: [String : AnyObject]?, context: UnsafeMutablePointer<Void>)
    {
        if let keyPath = keyPath
        {
            switch (keyPath, context)
            {
            case(self.dynamicType.ProgressKeyPath, &self.progressKVOContext):
                
                let progress = change?[NSKeyValueChangeNewKey]?.doubleValue ?? 0
                
                // Do something with progress                

            case(self.dynamicType.StateKeyPath, &self.stateKVOContext):
                
                let stateRaw = (change?[NSKeyValueChangeNewKey] as? String) ?? DescriptorState.Ready.rawValue;
                let state = DescriptorState(rawValue: stateRaw)!
                
                // Do something with state
                
            default:
                super.observeValueForKeyPath(keyPath, ofObject: object, change: change, context: context)
            }
        }
        else
        {
            super.observeValueForKeyPath(keyPath, ofObject: object, change: change, context: context)
        }
    }
```

Check out the `DescriptorKVObserver` class. It's a small utility that makes KVO'ing a Descriptor's `state` and `progress` properties easier.

### Canceling an Upload

Canceling an upload will cancel the file upload itself as well as delete the video object from Vimeo servers. You can cancel an upload using the `OldUploadDescriptor` instance in question:

```Swift
    let vimeoUpload = ... // Your instance of VimeoUpload (see above)
    let descriptor = ... // The descriptor you'd like to cancel
    
    vimeoUpload.cancelUpload(descriptor: descriptor)
```

Or by using the `identifier` of the `OldUploadDescriptor` in question: 

```Swift
    let vimeoUpload = ... // Your instance of VimeoUpload (see above)
    let identifier = phAsset.localIdentifier
    vimeoUpload.cancelUpload(identifier: identifier)
```

## Custom Workflows

TODO

## Found an Issue?

Please file it in the git [issue tracker](https://github.com/vimeo/VimeoUpload/issues).

## Want to Contribute?

If you'd like to contribute, please follow our guidelines found in [CONTRIBUTING.md](CONTRIBUTING.md).

## License

`VimeoUpload` is available under the MIT license. See the [LICENSE](LICENSE.md) file for more info.

## Questions?

Tweet at us here: [@vimeoapi](https://twitter.com/vimeoapi).

Post on [Stackoverflow](http://stackoverflow.com/questions/tagged/vimeo-ios) with the tag `vimeo-ios`.

Get in touch [here](https://vimeo.com/help/contact).

Interested in working at Vimeo? We're [hiring](https://vimeo.com/jobs)!
