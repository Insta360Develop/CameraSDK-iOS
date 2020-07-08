<img src="https://img.shields.io/badge/Platform-iOS(10.0, *)-blue"></img>
<img src="https://img.shields.io/badge/Version-2.6.9-blue"></img>
[![Carthage Compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)
[![OSC compatible](https://img.shields.io/badge/OSC-compatible-brightgreen)](ttps://developers.google.com/streetview/open-spherical-camera/reference)

# CameraSDK-iOS

You can learn how to control the insta360 camera in the following section

- Camera supports the control command of [Open Spherical Camera API level 2](https://developers.google.com/streetview/open-spherical-camera/reference), except preview stream. Familiarity with [`Open Spherical Camera API - Commands`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc/commands) official documentation is a prerequisite for OSC development.

## Table of Contents

- [Integration](#Integration)
	- [Carthage](#Carthage)
	- [Setup](#Setup)
- [Connection](#Connection)
- [Commands](#Commands)
- [Working with audio & video streams](#Audio_Video_Stream)
	- [Control center](#Control_center)
	- [Preview](#Preview)
	- [For further preview config](#Further_Config)
- [Medias](#Medias)
	- [INSExtraInfo](#INSExtraInfo)
	- [Thumbnail](#Thumbnail)
	- [Stitch](#Stitch)
	- [Generate HDR image](#Generate_HDR_image)
	- [Gyroscope data](#Gyroscope_data)
- [Internal parameters](#Internal_parameters)

## <a name="Integration" />Integration</a>

### <a name="Carthage" />Carthage</a>

Carthage is a decentralized dependency manager that builds your dependencies and provides you with binary frameworks. To integrate INSCameraSDK & INSCoreMedia into your Xcode project using Carthage, specify it in your Cartfile:

```ogdl
binary "https://ios-releases.insta360.com/INSCoreMedia.json" == 1.25.4
binary "https://ios-releases.insta360.com/INSCameraSDK-osc.json" == 2.6.9
```

### <a name="Setup" />Setup</a>

1. embed the INSCameraSDK and INSCoreMedia frameworks to your project target.
<div align=center><img src="./images/embedframework.png"/></div>

2. add an item in the Info.plist. Key is *Supported external accessory protocols*, value is an Array with following items `com.insta360.camera`(Nano), `com.insta360.onecontrol`(ONE), `com.insta360.onexcontrol`(ONE X), `com.insta360.nanoscontrol`(Nano S) and `com.insta360.onercontrol`(ONE R)
<div align=center><img src="./images/infoplist.png"/></div>

## <a name="Connection" />Connection</a>

If you connect camera via wifi, you need to set the host to `http://192.168.42.1`. and if you connect camera via the Lightning interface, you need to changed the host to `http://localhost:9099`.

We recommend that you use the following methods to convert of URL and path

```Objective-C
/// convert (photo or video) resource uri to http url via http tunnel and Wi-Fi socket
extern NSURL *INSHTTPURLForResourceURI(NSString *uri);

/// convert local http url to (photo or video) resource uri
extern NSString *INSResourceURIFromHTTPURL(NSURL *url);
```

### <a name="OSC" />OSC</a>

The camera already supports the [`/osc/info`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc/info) and [`/osc/state`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc/state) commands. You can use these commands to get basic information about the camera and the features it supports.

### <a name="INS_Protocol" />INS Protocol</a>

Add the following code in your AppDelegate, or somewhere your app is ready to work with Insta360 cameras via the Lightning interface. And if you're connected to the camera via WiFi, you should add `[[INSCameraManager socketManager] setup]` once where you need to start the socket connection. The connection is asynchronous. You need to monitor the connection status and operate the camera when the connection status is `INSCameraStateConnected`. What's more, call `[[INSCameraManager sharedManager] shutdown]` when your app won't listen on Insta360 cameras any more. [see connection monitoring](#Status)

```Objective-C
#import <INSCameraSDK/INSCameraSDK.h>

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [[INSCameraManager sharedManager] setup];
    return YES;
}
```

#### <a name="Status" />Status</a>

You can monitor the connection status of the camera in the following ways:

- register notification for `[NSNotificationCenter defaultCenter]` with the name of `INSCameraDidConnectNotification` or `INSCameraDidDisconnectNotification`

- add KVO on `[INSCameraManager SharedManager].cameraState`, once the cameraState changes to INSCameraStateConnected, your app is able to send commands to the camera.

#### <a name="Heartbeat" />Heartbeat</a>

When you connect your camera via wifi, you need to send heartbeat information to the camera at 2 Hz.

```Objective-C
// Objective-C
[[INSCameraManager socketManager].commandManager sendHeartbeatsWithOptions:nil]
```

## <a name="Commands" />Commands</a>

### <a name="Commands_OSC" />OSC</a>

Execute commands via Open Sepherial Camera API[`/osc/commands/execute`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc/commands/execute)

You need to poll yourself to call [`/osc/commands/status`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc/commands/status) to get the execution status of the camera on the current command. The polling cycle can be adjusted according to specific conditions. Here is a sample code shows how to execute a `camera.takePicture` command.

```Objective-C
#import <Foundation/Foundation.h>

NSDictionary *headers = @{ @"Content-Type": @"application/json",
                           @"X-XSRF-Protected": @"1",
                           @"Accept": @"application/json" };
NSDictionary *parameters = @{ @"name": @"camera.takePicture" };

NSData *postData = [NSJSONSerialization dataWithJSONObject:parameters options:0 error:nil];

NSURL *url = [NSURL URLWithString:@"#CameraHost#/commands/execute"];
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url
                                                       cachePolicy:NSURLRequestUseProtocolCachePolicy
                                                   timeoutInterval:10.0];
[request setHTTPMethod:@"POST"];
[request setAllHTTPHeaderFields:headers];
[request setHTTPBody:postData];

NSURLSession *session = [NSURLSession sharedSession];
[[session dataTaskWithRequest:request
            completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
    if (error) {
        NSLog(@"%@", error);
    } else {
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *) response;
        NSLog(@"%@", httpResponse);
    }
}] resume];
```

#### <a name="OSC_Options" />Options</a>

* You can use [`camera.setOptions`](https://developers.google.com/streetview/open-spherical-camera/reference/camera/setoptions) to set options.
* You can use [`camera.getOptions`](https://developers.google.com/streetview/open-spherical-camera/reference/camera/getoptions) to get options.
* You can know the relevant parameters supported by the camera from [Open Spherical Camera API options](https://developers.google.com/streetview/open-spherical-camera/reference/options).

#### <a name="OSC_Take_picture_Record" />Take picture & Record</a>

* You can use [`camera.takePicture`](https://developers.google.com/streetview/open-spherical-camera/reference/camera/takepicture) to take picture.
* You can use [`camera.startCapture`](https://developers.google.com/streetview/open-spherical-camera/reference/camera/startcapture) to start record.
* You can use [`camera.stopCapture `](https://developers.google.com/streetview/open-spherical-camera/reference/camera/stopcapture) to stop record.

#### <a name="OSC_List_files" />List files</a>

You can use [`camera.listFiles`](https://developers.google.com/streetview/open-spherical-camera/reference/camera/listfiles) to get the files list.

### <a name="Commands_INS_Protocol" />INS Protocol</a>

You can learn all the commands supported by the SDK from `INSCameraCommands.h`, and `INSCameraCommandOptions.h` shows the structure needed by all commands. All options that you can get from camera are list in `INSCameraOptionsType`.

`NSNotification+INSCamera.h` show that the application can get the notification of camera by monitoring.

Here is a sample code shows how to get storage & battery through `getOptionsWithTypes:completion:`

```Objective-C
NSArray *optionTypes = @[@(INSCameraOptionsTypeStorageState),@(INSCameraOptionsTypeBatteryStatus)];
[[INSCameraManager sharedManager].commandManager getOptionsWithTypes:optionTypes completion:^(NSError * _Nullable error, INSCameraOptions * _Nullable options, NSArray<NSNumber *> * _Nullable successTypes) {
    if (!options) {
        NSLog(@"fetch options error: %@",error.description);
        return ;
    }
    NSLog(@"storage status: %@",options.storageStatus);
    NSLog(@"battery status: %@",options.batteryStatus);
}];
```

## <a name="Audio_Video_Stream" />Working with audio & video streams</a>

Audio and video stream is based on INS protocol. If you need to preview the camera in real time, make sure that the `INSCameraManager.cameraState` is `INSCameraStateConnected`. see [INS Protocol connection](#INS_Protocol)

### <a name="Control_center" />Control center - `INSCameraMediaSession`</a>

`INSCameraMediaSession` is the central class to work with audio & video streams for Nano or ONE camera. It has these functions:

1. You can configure the input (the camera) by set the `expectedAudioSampleRate`, `expectedVideoResolution` and `gyroPlayMode`.
2. Control the camera's input streams, turn on by calling `startRunningWithCompletion:`, turn off by call `stopRunningWithCompletion:`.
3. Parse an, decode media data, stitch the video.
4. Distribute outputs to `INSCameraMediaPluggable` such as `INSCameraFlatPanoOutput`.
5. When the session is running, you can change the input configurations, plug or unplug pluggables, make the changes working by call `commitChangesWithCompletion:`.

### <a name="Preview" />Preview</a>

```Objective-C
#import <UIKit/UIKit.h>
#import <INSCameraSDK/INSCameraSDK.h>

@interface ViewController () <INSCameraPreviewPlayerDelegate>

@property (nonatomic, strong) INSCameraMediaSession *mediaSession;

@property (nonatomic, strong) INSCameraPreviewPlayer *previewPlayer;

@property (nonatomic, strong) INSCameraStorageStatus *storageState;

@property (nonatomic, assign) INSVideoEncode videoEncode;

@end

@implementation ViewController

- (void)dealloc {
    [_mediaSession stopRunningWithCompletion:^(NSError * _Nullable error) {
        NSLog(@"stop media session with err: %@", error);
    }];
    
    [[INSCameraManager usbManager] removeObserver:self forKeyPath:@"cameraState"];
    [[INSCameraManager socketManager] removeObserver:self forKeyPath:@"cameraState"];
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [[INSCameraManager usbManager] addObserver:self forKeyPath:@"cameraState" options:NSKeyValueObservingOptionNew context:nil];
    [[INSCameraManager socketManager] addObserver:self forKeyPath:@"cameraState" options:NSKeyValueObservingOptionNew context:nil];
    
    [self setupRenderView];
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    
    if ([INSCameraManager sharedManager].currentCamera) {
        __weak typeof(self)weakSelf = self;
        [self fetchOptionsWithCompletion:^{
            [weakSelf updateConfiguration];
            [weakSelf runMediaSession];
        }];
    }
}

- (void)updateConfiguration {
    _mediaSession.expectedVideoResolution = _configurationVC.inputVideoResolution;
    _mediaSession.expectedVideoResolutionSecondary = _configurationVC.inputVideoResolution2;
    _mediaSession.previewStreamType = INSPreviewStreamTypeWithValue(_configurationVC.previewStreamNum);
    _mediaSession.expectedAudioSampleRate = _configurationVC.audioSampleRate;
    _mediaSession.videoStreamEncode = _videoEncode;
    
    XLFormRowDescriptor *row = [self.form formRowWithTag:@"Gyro Statiblity On"];
    _mediaSession.gyroPlayMode = [row.value boolValue] ? _configurationVC.gyroPlayMode : INSGyroPlayModeNone;
}

- (void)setupRenderView {
    CGFloat height = CGRectGetHeight(self.view.bounds) * 0.333;
    CGRect frame = CGRectMake(0, CGRectGetHeight(self.view.bounds) - height, CGRectGetWidth(self.view.bounds), height);
    _previewPlayer = [[INSCameraPreviewPlayer alloc] initWithFrame:frame
                                                        renderType:INSRenderTypeSphericalPanoRender];
    [_previewPlayer playWithGyroTimestampAdjust:30.f];
    _previewPlayer.delegate = self;
    [self.view addSubview:_previewPlayer.renderView];
    
    [_mediaSession plug:self.previewPlayer];
    
    // adjust field of view parameters
    NSString *offset = [INSCameraManager sharedManager].currentCamera.settings.mediaOffset;
    if (offset) {
        NSInteger rawValue = [[INSLensOffset alloc] initWithOffset:offset].lensType;
        if (rawValue == INSLensTypeOneR577Wide || rawValue == INSLensTypeOneR283Wide) {
            _previewPlayer.renderView.enablePanGesture = NO;
            _previewPlayer.renderView.enablePinchGesture = NO;
            
            _previewPlayer.renderView.render.camera.xFov = 37;
            _previewPlayer.renderView.render.camera.distance = 700;
        }
    }
}

- (void)fetchOptionsWithCompletion:(nullable void (^)(void))completion {
    __weak typeof(self)weakSelf = self;
    NSArray *optionTypes = @[@(INSCameraOptionsTypeStorageState),@(INSCameraOptionsTypeVideoEncode)];
    [[INSCameraManager sharedManager].commandManager getOptionsWithTypes:optionTypes completion:^(NSError * _Nullable error, INSCameraOptions * _Nullable options, NSArray<NSNumber *> * _Nullable successTypes) {
        if (!options) {
            [weakSelf showAlertWith:@"Get options" message:error.description];
            completion();
            return ;
        }
        weakSelf.storageState = options.storageStatus;
        weakSelf.videoEncode = options.videoEncode;
        completion();
    }];
}

- (void)runMediaSession {
    if ([INSCameraManager sharedManager].cameraState != INSCameraStateConnected) {
        return ;
    }
    
    __weak typeof(self)weakSelf = self;
    if (_mediaSession.running) {
        self.view.userInteractionEnabled = NO;
        [_mediaSession commitChangesWithCompletion:^(NSError * _Nullable error) {
            NSLog(@"commitChanges media session with error: %@",error);
            weakSelf.view.userInteractionEnabled = YES;
            if (error) {
                [weakSelf showAlertWith:@"commitChanges media failed" message:error.description];
            }
        }];
    }
    else {
        self.view.userInteractionEnabled = NO;
        [_mediaSession startRunningWithCompletion:^(NSError * _Nullable error) {
            NSLog(@"start running media session with error: %@",error);
            weakSelf.view.userInteractionEnabled = YES;
            if (error) {
                [weakSelf showAlertWith:@"start media failed" message:error.description];
                [weakSelf.previewPlayer playWithSmoothBuffer:NO];
            }
        }];
    }
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    if ([keyPath isEqualToString:@"cameraState"]) {
        INSCameraState state = [change[NSKeyValueChangeNewKey] unsignedIntegerValue];
        switch (state) {
            case INSCameraStateFound:
                break;
            case INSCameraStateConnected:
                [self runMediaSession];
                break;
            default:
                [_mediaSession stopRunningWithCompletion:nil];
                break;
        }
    }
}

#pragma mark INSCameraPreviewPlayerDelegate
- (NSString *)offsetToPlay:(INSCameraPreviewPlayer *)player {
    NSString *mediaOffset = [INSCameraManager sharedManager].currentCamera.settings.mediaOffset;
    if (([[INSCameraManager sharedManager].currentCamera.name isEqualToString:kInsta360CameraNameOneX]
         || [[INSCameraManager sharedManager].currentCamera.name isEqualToString:kInsta360CameraNameOneR]
         || [[INSCameraManager sharedManager].currentCamera.name isEqualToString:kInsta360CameraNameOneX2])
        && [INSLensOffset isValidOffset:mediaOffset]) {
        return [INSOffsetCalculator convertOffset:mediaOffset toType:INSOffsetConvertTypeOneX3040_2_2880];
    }
    
    return mediaOffset;
}

@end
```

### <a name="Further_Config" />For further preview config</a>

* You can configure the preview resolution through the following parameters of `INSCameraMediaSession`, and all supported resolutions are list in `INSCameraMediaBasic.h`.

```Objective-C
/*!
 *  For one、nano s,  The expected video resolution, if you want to change the value when mediaSession is running, you need to invoke commitChangesWithCompletion:
 *  For one x, you should set resolution for both Main and Secondary stream. use 'INSPreviewStreamType' to choose which is used for preview stream.
 */
@property (nonatomic) INSVideoResolution expectedVideoResolution;

/*!
 *  The expected video resolution, if you want to change the value when mediaSession is running, you need to invoke commitChangesWithCompletion:
 */
@property (nonatomic) INSVideoResolution expectedVideoResolutionSecondary;

/*!
 *  For one X, use this to choose whether the main or secondary stream should be used as preview stream.
 *  INSPreviewStreamTypeMain : preview with main stream
 *  INSPreviewStreamTypeSecondary : preview with secondary stream
 */
@property (nonatomic) INSPreviewStreamType previewStreamType;

/*!
 *  VR180 and gyroPlayMode == RemoveYawRotations, this should be set to INSPreviewStreamRotationRaw180
 */
@property (nonatomic) INSPreviewStreamRotation previewStreamRotation;

/*!
 * The mode used to calibrate the video, default is INSGyroPlayModeDefault.
 */
@property (nonatomic) INSGyroPlayMode gyroPlayMode;

/*!
 *  The encoding format of video real-time stream, default is INSVideoEncodeH264.
 */
@property (nonatomic) INSVideoEncode videoStreamEncode;
```

## <a name="Medias" />Medias</a>

There is a special data segment in the video or photo captured by Insta360 camera, which is called INSExtraInfo. The INSExtraInfo contains the corresponding file's thumbnail, extra metedata, gyroscope data, etc. For more information, please check `INSExtraInfo.h`.

In general, we suggests that you obtain the above information from a file whose file name `(VIN Channel)(Stream Num)` is '00'. For example: `IMG_19700101_000000_00_001.insp`.

If you are working on a wide angle file, and the file is a selfies file ( [how to knonw a file is a selfies file](#INSExtraInfo) ), you should use `IMG_19700101_000000_10_001.insp` which `(VIN Channel)` is '1' instead .

- fileName format: `(IMG/VID)_Date_Time_(VIN Channel)(Stream Num)_Serial.Extension`

### <a name="INSExtraInfo" />INSExtraInfo</a>

INSExtraInfo contains the corresponding file's thumbnail, extra metedata, gyroscope data, etc. You can get the above information through `INSImageInfoParser`/`INSVideoInfoParser`.

```Objective-C
NSURL *url = #source url#
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:url];
if ([parser open]) {
    BOOL isLensSelfies = parser.extraInfo.metadata.lensSelfies;
}
```

```Objective-C
NSURL *url = #source url#
INSVideoInfoParser *parser = [[INSVideoInfoParser alloc] initWithURL:url];
if ([parser openFast]) {
    BOOL isLensSelfies = parser.extraInfo.metadata.lensSelfies;
}
```

### <a name="Thumbnail" />Thumbnail</a>

#### <a name="Thumbnail_Photos" />Photos</a>

You can get the pre stored thumbnail data in the file through `INSImageInfoParser`. 

```Objective-C
NSURL *url = #source url#
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:url];
if ([parser open]) {
    NSData *data = parser.extraInfo.thumbnail;
    UIImage *thumbnail = [[UIImage alloc] initWithData:data];
}
```
If you are working on a panoramic file, you also need to stitch the file, see [how to stitch image](#Stitch).

#### <a name="Thumbnail_Videos" />Videos</a>

You can get the pre stored thumbnail data in the file through `INSVideoInfoParser`. 

```Objective-C
NSURL *url = #source url#
INSVideoInfoParser *parser = [[INSVideoInfoParser alloc] initWithURL:url];

UIImage *thumbnail;
if ([parser openFast]) {
    INSThumbnailRender *render = [[INSThumbnailRender alloc] init];
    
    NSData *data = parser.extraInfo.thumbnail;
    CVPixelBufferRef buffer = [render copyPixelBufferWithDecodeH264Data:data];
    
    // if and only if the video resolution is 5.7k, the thumbnails are divided into thumbnail and ext_thumbnail
    if (parser.extraInfo.metadata.dimension.width == INSVideoResolution2880x2880x30.width
        && parser.extraInfo.metadata.dimension.height == INSVideoResolution2880x2880x30.height) {
        NSData *extData = parser.extraInfo.ext_thumbnail;
        CVPixelBufferRef extBuffer = [render copyPixelBufferWithDecodeH264Data: extData];
        thumbnail = [UIImage imageWithPixelBuffer:buffer rightPixelBuffer:extBuffer];
    } else {
        thumbnail = [UIImage imageWithPixelBuffer:buffer];
    }
}
```
If you are working on a panoramic file, you also need to do a splicing of the file, see [how to stitch image](#Stitch).

### <a name="Stitch" />Stitch</a>

The following parameters are needed to correct and stitch a picture:

- offset
- gyroscope data
- resolution

Here is a sample code shows how to get these parameters:

```Objective-C 
#pragma mark Photos

NSURL *url = #source url#
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:url];
if ([parser open]) {
    // offset
    NSString *offset = parser.extraInfo.metadata.offset;

    // resolution
    CGSize resolution = parser.extraInfo.metadata.dimension;

    // gyroscope data
    INSExtraGyroData *gyroData = parser.extraInfo.gyroData;
}
```

```Objective-C 
#pragma mark Videos

NSURL *url = #source url#
INSVideoInfoParser *parser = [[INSVideoInfoParser alloc] initWithURL:url];
if ([parser openFast]) {
    // offset
    NSString *offset = parser.extraInfo.metadata.offset;

    // resolution
    CGSize resolution = parser.extraInfo.metadata.dimension;

    // gyroscope data
    INSExtraGyroData *gyroData = parser.extraInfo.gyroData;
}
```

#### <a name="Stitch_Photos" />Photos</a>

Using `INSFlatPanoOffscreenRender` to get a flat pano image. ( P.s. The parameter, `offset` is nonnull )

```Objective-C 
// if it is the original image, we recommend that outputsize be set to `parser.extraInfo.metadata.dimension`
CGSize outputSize = #output size#
UIImage *origin = #photo thumbnail to be stitched#

NSURL *url = #source url#
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:url];
if ([parser open]) {
    INSFlatPanoOffscreenRender *render = [[INSFlatPanoOffscreenRender alloc] initWithRenderWidth:outputSize.width height:outputSize.height];
    render.eulerAdjust = parser.extraInfo.metadata.euler;
    render.offset = parser.extraInfo.metadata.offset;
    
    render.gyroStabilityOrientation = GLKQuaternionIdentity;
    if (parser.extraInfo.gyroData) {
        INSGyroPBPlayer *gyroPlayer = [[INSGyroPBPlayer alloc] initWithPBGyroData:parser.extraInfo.gyroData];
        GLKQuaternion orientation = [gyroPlayer getImageOrientationWithRenderType:INSRenderTypeFlatPanoRender];
        render.gyroStabilityOrientation = orientation;
    }
    
    [render setRenderImage:origin];
    UIImage *output = [render renderToImage];
}
```

#### <a name="Stitch_Videos" />Videos</a>

Using `INSFlatPanoOffscreenRender` to get a flat pano image. ( P.s. The parameter, `offset` is nonnull )

```Objective-C 
// if it is the original image, we recommend that outputsize be set to `parser.extraInfo.metadata.dimension`
CGSize outputSize = #output size#
CVPixelBufferRef buffer = #video thumbnail to be stitched#

// if and only if the video resolution is 5.7k, the thumbnails are divided into thumbnail and ext_thumbnail
CVPixelBufferRef extBuffer = #video thumbnail to be stitched#

NSURL *url = #source url#
INSVideoInfoParser *parser = [[INSVideoInfoParser alloc] initWithURL:url];
if ([parser openFast]) {
    INSFlatPanoOffscreenRender *render = [[INSFlatPanoOffscreenRender alloc] initWithRenderWidth:outputSize.width height:outputSize.height];
    render.eulerAdjust = parser.extraInfo.metadata.euler;
    render.offset = parser.extraInfo.metadata.offset;
    
    render.gyroStabilityOrientation = GLKQuaternionIdentity;
    if (parser.extraInfo.gyroData) {
        INSGyroPBPlayer *gyroPlayer = [[INSGyroPBPlayer alloc] initWithPBGyroData:parser.extraInfo.gyroData];
        GLKQuaternion orientation = [gyroPlayer getImageOrientationWithRenderType:INSRenderTypeFlatPanoRender];
        render.gyroStabilityOrientation = orientation;
    }
    
    if (buffer && extBuffer) {
        [render setRenderPixelBuffer:buffer right:extBuffer timestamp:parser.extraInfo.metadata.thumbnailGyroTimestamp];
    } else if (buffer) {
        [render setRenderPixelBuffer:buffer timestamp:parser.extraInfo.metadata.thumbnailGyroTimestamp];
    }
    UIImage *output = [render renderToImage];
}
```

### <a name="Generate_HDR_image" />Generate HDR image</a>

Using `INSHDRTask` to generate HDR image. HDR synthesis takes a long time and takes about 5-10 seconds. 

The URLs that is passed into `INSHDROptions` is ordered array, and the array order is [ 0ev, -ev, +ev ]. Through the photos taken by ONE X, the default ascending order of the file name is [ 0ev, -ev, +ev ].

```Objective-C
NSArray *urls = @[
    [NSURL URLWithString:@"0ev"],
    [NSURL URLWithString:@"-ev"],
    [NSURL URLWithString:@"+ev"],
];

INSHDROptions *options = [[INSHDROptions alloc] init];
options.urls = urls;
options.seamlessType = INSSeamlessTypeOpticalFlow;

INSHDRTask *task = [[INSHDRTask alloc] initWithCommandManager:[INSCameraManager sharedManager].commandManager];
[task processWithOptions:options completion:^(NSError * _Nullable error, NSData * _Nullable photoData) {
    if (error) {
        NSLog(@"%@ failed with error: %@",sender.title, error);
        return ;
    }
    
    // do anything with the stitched image here, for example, display it
    if (photoData) {
        UIImage *image = [[UIImage alloc] initWithData:photoData];
        UIImageWriteToSavedPhotosAlbum(image, nil, nil, nil);
    }
}];
```

You can choose the following two lib for HDR synthesis, and the preferred lib is `INSHDRLibInsImgProc`.

```Objective-C
typedef NS_ENUM(NSUInteger, INSHDRLib) {
    /// using `OpenCV` to generate hdr image
    INSHDRLibOpenCV,
    
    /// using `InsImgProLib` to generate hdr image
    INSHDRLibInsImgProc,
};
```

You can choose the following two algorithms for HDR synthesis, and the preferred algorithms is `INSSeamlessTypeOpticalFlow`.

```Objective-C
typedef NS_ENUM(NSUInteger, INSSeamlessType) {
    /// default type
    INSSeamlessTypeTemplate,
    
    /// using Optical flow
    INSSeamlessTypeOpticalFlow,
};
```

### <a name="Gyroscope_data" />Gyroscope data - `INSMediaGyro`</a>

* You can get the gyroscope data(`INSMediaGyro`) of `INSExtraInfo.metadata.thumbnailGyroTimestamp` through `INSImageInfoParser`. `INSMediaGyro` shows the `ax, ay, az, gx, gy, gz` information of the gyroscope.

```Objective-C
NSURL *url = #source url#
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:url];
if ([parser open]) {
	NSLog(@"%@",parser.extraInfo.metadata.gyro);
}
```

* You can get the complete gyroscope data in the following ways:

```Objective-C
NSURL *url = #source url#
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:url];
if ([parser open]) {
	NSLog(@"%@",parser.extraInfo.gyroData);
}
```

#### <a name="Media_gyro_ajust" />Media gyro ajust</a>

You can use `INSFlatGyroAdjustOffscreenRender` to correct the stitched image:

```Objective-C 
// if it is the original image, we suggests that using `parser.extraInfo.metadata.dimension`
CGSize outputSize = #output size#
UIImage *origin = #photo thumbnail to be stitched#

NSURL *url = #source url#
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:url];
if ([parser open]) {
    CGSize outputSize = parser.extraInfo.metadata.dimension;
    INSFlatGyroAdjustOffscreenRender *render = [[INSFlatGyroAdjustOffscreenRender alloc] initWithRenderWidth:outputSize.width height:outputSize.height];
    render.eulerAdjust = parser.extraInfo.metadata.euler;
    render.offset = parser.extraInfo.metadata.offset;
    
    render.gyroStabilityOrientation = GLKQuaternionIdentity;
    if (parser.extraInfo.gyroData) {
        INSGyroPBPlayer *gyroPlayer = [[INSGyroPBPlayer alloc] initWithPBGyroData:parser.extraInfo.gyroData];
        GLKQuaternion orientation = [gyroPlayer getImageOrientationWithRenderType:INSRenderTypeFlatPanoRender];
        render.gyroStabilityOrientation = orientation;
    }
    
    [render setRenderImage:origin];
    UIImage *output = [render renderToImage];
}
```

## <a name="Internal_parameters" />Internal parameters</a>

* Insta360 fisheye distortion:

<div align=left><img src="./images/INSFisheyeDistortion.png"/></div>

* Opencv fisheye distortion:

<div align=left><img src="./images/OpenCVFisheyeDistortion.png"/></div>

Using `INSOffsetParser` to get the `INSOffsetParameter` internal parameters

```Objective-C
NSURL *url = #source url#
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:url];
if ([parser open]) {
    INSExtraInfo *extraInfo = parser.extraInfo;
    
    INSOffsetParser *offsetParser =
    [[INSOffsetParser alloc] initWithOffset:parser.extraInfo.metadata.offset
                                      width:extraInfo.metadata.dimension.width
                                     height:extraInfo.metadata.dimension.height];
    for (INSOffsetParameter *param in offsetParser.parameters) {
        NSLog(@"Internal parameters: %@", param);
    }
}
```
