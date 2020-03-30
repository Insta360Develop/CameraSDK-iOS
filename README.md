# CameraSDK-iOS
iOS SDK to control Insta360 cameras.

### Integration

1. embed the INSCameraSDK and INSCoreMedia frameworks to your project target.
![embedframework](./images/embedframework.png)

2. add an item in the Info.plist. Key is *Supported external accessory protocols*, value is an Array with 3 items `com.insta360.camera`(Nano), `com.insta360.onecontrol`(ONE), `com.insta360.onexcontrol`(ONE X) and `com.insta360.nanoscontrol`(Nano S)
![infoplist](./images/infoplist.png)

3. Add the following code in your AppDelegate, or somewhere your app is ready to work with Insta360 cameras.

```objc
// Objective-C
#import <INSCameraSDK/INSCameraSDK.h>

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [[INSCameraManager sharedManager] setup];
    return YES;
}
```

4. Call `[[INSCameraManager sharedManager] shutdown]` when your app won't listen on Insta360 cameras any more.

### Monitor Connection of Insta360 Nano, ONE, Nano S Cameras

- register notification for `[NSNotificationCenter defaultCenter]` with the name of `INSCameraDidConnectNotification` or `INSCameraDidDisconnectNotification`

- you can also add KVO on `[INSCameraManager SharedManager].cameraState`, once the cameraState changes to INSCameraStateConnected, your app is able to send commands to the camera.

#### Heartbeats

- When you connect your camera via wifi, you need to send heartbeat information to the camera at 2 Hz.

```objc
// Objective-C
[[INSCameraManager socketManager].commandManager sendHeartbeatsWithOptions:nil]

```

### Send commands

Familiarity with [`Open Spherical Camera API - Commands`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc/commands) official documentation is a prerequisite for OSC development.

The camera network address is `http://192.168.42.1`.Execute commands via Open Sepherial Camera API[`/osc/commands/execute`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc/commands/execute)

You need to poll yourself to call [`/osc/commands/status`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc/commands/status) to get the execution status of the camera on the current command. The polling cycle can be adjusted according to specific conditions.

Camera support [Open Spherical Camera API level 2](https://developers.google.com/streetview/open-spherical-camera/reference), except preview stream.

#### Connecting with Lightning Interface

When connecting a camera with the Lightning interface, the camera address needs to be changed to `http://localhost:9099`

```Objective-C
#import <Foundation/Foundation.h>

NSDictionary *headers = @{ @"Content-Type": @"application/json",
                           @"X-XSRF-Protected": @"1",
                           @"Accept": @"application/json" };
NSDictionary *parameters = @{ @"name": @"camera.takePicture" };

NSData *postData = [NSJSONSerialization dataWithJSONObject:parameters options:0 error:nil];

NSURL *url = [NSURL URLWithString:@"http://localhost:9099/osc/commands/execute"];
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url
                                                       cachePolicy:NSURLRequestUseProtocolCachePolicy
                                                   timeoutInterval:10.0];
[request setHTTPMethod:@"POST"];
[request setAllHTTPHeaderFields:headers];
[request setHTTPBody:postData];

NSURLSession *session = [NSURLSession sharedSession];
[[session dataTaskWithRequest:[[NSURLRequest alloc] init]
            completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
    if (error) {
        NSLog(@"%@", error);
    } else {
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *) response;
        NSLog(@"%@", httpResponse);
    }
}] resume];
```

#### Take picture & Record

* You can use [`camera.takePicture`](https://developers.google.com/streetview/open-spherical-camera/reference/camera/takepicture) to take picture.
* You can use [`camera.startCapture`](https://developers.google.com/streetview/open-spherical-camera/reference/camera/startcapture) to start record.
* You can use [`camera.stopCapture `](https://developers.google.com/streetview/open-spherical-camera/reference/camera/stopcapture) to stop record.

#### Options

* You can use [`camera.setOptions`](https://developers.google.com/streetview/open-spherical-camera/reference/camera/setoptions) to set options.
* You can use [`camera.getOptions`](https://developers.google.com/streetview/open-spherical-camera/reference/camera/getoptions) to get options.
* You can know the relevant parameters supported by the camera from [Open Spherical Camera API options](https://developers.google.com/streetview/open-spherical-camera/reference/options).

#### List files

You can use [`camera.listFiles`](https://developers.google.com/streetview/open-spherical-camera/reference/camera/listfiles) to get the files list.

### Status & Informations

#### 1. INSCameraSDK

Not only your app can send commands to the camera, but also the app will receive some notifications from camera when some events happen. For example the batter status changes.
All notifications are listed in `NSNotification+INSCamera.h` file. 

#### 2. Open Spherical Camera API

Familiarity with [`Open Spherical Camera API`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc) official documentation is a prerequisite for OSC development.

* You can use [`/osc/info`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc/info) to get the basic information about the camera and functionality it supports.
* You can use [`/osc/state`](https://developers.google.cn/streetview/open-spherical-camera/guides/osc/state) to get the attributes of the camera.

### Working with audio & video streams

#### Control center - `INSCameraMediaSession`

`INSCameraMediaSession` is the central class to work with audio & video streams for Nano or ONE camera. It has these functions:

1. You can configure the input (the camera) by set the `expectedAudioSampleRate`, `expectedVideoResolution` and `gyroPlayMode`.
2. Control the camera's input streams, turn on by calling `startRunningWithCompletion:`, turn off by call `stopRunningWithCompletion:`.
3. Parse an, decode media data, stitch the video.
4. Distribute outputs to `INSCameraMediaPluggable` such as `INSCameraFlatPanoOutput`.
5. When the session is running, you can change the input configurations, plug or unplug pluggables, make the changes working by call `commitChangesWithCompletion:`.

#### Preview

```objc
#import <UIKit/UIKit.h>
#import <INSCameraSDK/INSCameraSDK.h>

@interface ViewController () <INSCameraPreviewPlayerDelegate>

@property (nonatomic, strong) INSCameraMediaSession *mediaSession;

@property (nonatomic, strong) INSCameraPreviewPlayer *previewPlayer;

@property (nonatomic, strong) INSCameraStorageStatus *storageState;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _mediaSession = [[INSCameraMediaSession alloc] init];

    _previewPlayer = [[INSCameraPreviewPlayer alloc] initWithFrame:self.view.bounds
                                                        renderType:INSRenderTypeSphericalPanoRender];
    [_previewPlayer playWithGyroTimestampAdjust:30.f];
    _previewPlayer.delegate = self;
    [self.view addSubview:_previewPlayer.renderView];
    
    [_mediaSession plug:self.previewPlayer];
    
    _mediaSession.expectedVideoResolution = INSVideoResolution3840x1920x30;
    _mediaSession.expectedVideoResolutionSecondary = INSVideoResolution960x480x30;
    _mediaSession.previewStreamType = INSPreviewStreamTypeSecondary;
    _mediaSession.expectedAudioSampleRate = INSAudioSampleRate48000Hz;
    _mediaSession.gyroPlayMode = INSGyroPlayModeDefault;
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    
    __weak typeof(self)weakSelf = self;
    [_mediaSession stopRunningWithCompletion:^(NSError * _Nullable error) {
        [weakSelf runMediaSession];
    }];
}

#pragma mark INSCameraPreviewPlayerDelegate
- (NSString *)offsetToPlay:(INSCameraPreviewPlayer *)player {
    NSString *mediaOffset = [INSCameraManager sharedManager].currentCamera.settings.mediaOffset;
    if ([[INSCameraManager sharedManager].currentCamera.name isEqualToString:kInsta360CameraNameOneX]
        && [INSLensOffset isValidOffset:mediaOffset]) {
        return [INSOffsetCalculator convertOffset:mediaOffset toType:INSOffsetConvertTypeOneX3040_2_2880];
    }
    
    return mediaOffset;
}

@end
```

### Stitch & HDR

#### Thumbnail

Retrieve thumbnail data from pre-existing files by `INSImageInfoParser` which resolution is 1920 * 960. And then using `INSFlatPanoOffscreenRender` to generate the thumbnail.

You can get thumbnail render and configure output size via `[[INSFlatPanoOffscreenRender alloc] initWithRenderWidth: height:]`

```objc
// [ev0, -ev, +ev]
NSMutableArray<NSURL *> *urls = [[NSMutableArray alloc] init];
NSArray *fileNames = @[@"IMG_20181029_182547_00_295", @"IMG_20181029_182547_00_296", @"IMG_20181029_182547_00_297"];
for (NSString *fileName in fileNames) {
    NSString *path = [[NSBundle mainBundle] pathForResource:fileName ofType:@"jpg"];
    NSURL *url = [NSURL fileURLWithPath:path];
    [urls addObject:url];
}

INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:urls.firstObject];
if ([parser open]) {
    NSData *data = parser.extraInfo.thumbnail;
    UIImage *thumbnail = [[UIImage alloc] initWithData:data];
    
    CGSize outputSize = CGSizeMake(CGRectGetWidth(self.view.bounds), CGRectGetWidth(self.view.bounds) / 2);
    UIImage *output = [weakSelf stitchImage:thumbnail extraInfo:parser.extraInfo outputSize:outputSize];
}
```

#### Stitch

Using `INSImageInfoParser` to get the gyroscope data, offset, resolution, etc.

Using `INSFlatPanoOffscreenRender` to get a flat pano image. ( P.s. The parameter, `offset` is nonnull )

```objc 
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:urls.firstObject];
if ([parser open]) {
    UIImage *origin = [[UIImage alloc] initWithContentsOfFile:urls.firstObject.path];
    
    CGSize outputSize = parser.extraInfo.metadata.dimension;
    INSFlatPanoOffscreenRender *render = [[INSFlatPanoOffscreenRender alloc] initWithRenderWidth:outputSize.width height:outputSize.height];
	render.eulerAdjust = extraInfo.metadata.euler;
	render.offset = extraInfo.metadata.offset;
	
	render.gyroStabilityOrientation = GLKQuaternionIdentity;
	if (extraInfo.gyroData) {
	    INSGyroPBPlayer *gyroPlayer = [[INSGyroPBPlayer alloc] initWithPBGyroData:extraInfo.gyroData];
	    GLKQuaternion orientation = [gyroPlayer getImageOrientationWithRenderType:INSRenderTypeFlatPanoRender];
	    render.gyroStabilityOrientation = orientation;
	}
	
	UIImage *output = [render setRenderImage:image];
}
```

### Generate HDR image

Using `INSHDRTask` to generate HDR image. HDR synthesis takes a long time and takes about 5-10 seconds. 

The URLs that is passed into `INSHDROptions` is ordered array, and the array order is [ ev0, -ev, +ev ]. Through the photos taken by ONE X, the default ascending order of the file name is [ ev0, -ev, +ev ].

```objc
INSHDRTask *task = [[INSHDRTask alloc] initWithCommandManager:[INSCameraManager sharedManager].commandManager];
[task processWithURLs:urls seamlessType:INSSeamlessTypeOpticalFlow completion:^(NSError * _Nullable error, INSHDRImageDataModel * _Nullable dataModel) {
    if (error) {
        NSLog(@"%@ failed with error: %@",sender.title, error);
        return ;
    }
    
    // do anything with the HDR Mat rgb data, for example, stitched image here
    NSString *documentPath = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
    NSURL *url = [[NSURL fileURLWithPath:documentPath] URLByAppendingPathComponent:@"HDRDataRaw"];
    [dataModel.data writeToURL:url atomically:YES];
}];
```

The HDR synthesis algorithm library is divided into two types: ONE X recommends `INSHDRLibInsImgProc`.

```objc
typedef NS_ENUM(NSUInteger, INSHDRLib) {
    /// using `OpenCV` to generate hdr image
    INSHDRLibOpenCV,
    
    /// using `InsImgProLib` to generate hdr image
    INSHDRLibInsImgProc,
};
```

The following two stitching algorithms are encapsulated in HDR synthesis process:

```objc
typedef NS_ENUM(NSUInteger, INSSeamlessType) {
    /// default type
    INSSeamlessTypeTemplate,
    
    /// using Optical flow
    INSSeamlessTypeOpticalFlow,
};
```

### Gyroscope data - `INSMediaGyro`

* You can get the `INSMediaGyro` which contains `ax, ay, az, gx, gy, gz` via `INSImageInfoParser`

```objc
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:urls.firstObject];
if ([parser open]) {
	NSLog(@"%@",parser.gyroData);
}
```

* If there is an `INSExtraInfo` instance, `INSMediaGyro` can be directly obtained.

```objc
INSMediaGyro *gyro = extraInfo.metadata.gyro;
NSLog(@"%@", gyro);
```

#### Media gyro ajust

Gyroscopic correction of 2:1 planar images that have been stitched can be done by `INSFlatGyroAdjustOffscreenRender`:

```objc 
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:urls.firstObject];
if ([parser open]) {
    UIImage *origin = [[UIImage alloc] initWithContentsOfFile:urls.firstObject.path];
    
    CGSize outputSize = parser.extraInfo.metadata.dimension;
    INSFlatGyroAdjustOffscreenRender *render = [[INSFlatGyroAdjustOffscreenRender alloc] initWithRenderWidth:outputSize.width height:outputSize.height];
	render.eulerAdjust = extraInfo.metadata.euler;
	render.offset = extraInfo.metadata.offset;
	
	render.gyroStabilityOrientation = GLKQuaternionIdentity;
	if (extraInfo.gyroData) {
	    INSGyroPBPlayer *gyroPlayer = [[INSGyroPBPlayer alloc] initWithPBGyroData:extraInfo.gyroData];
	    GLKQuaternion orientation = [gyroPlayer getImageOrientationWithRenderType:INSRenderTypeFlatPanoRender];
	    render.gyroStabilityOrientation = orientation;
	}
	
	UIImage *output = [render setRenderImage:image];
}
```

### Internal parameters

* Insta360 fisheye distortion:

![INSFisheyeDistortion](./images/INSFisheyeDistortion.png)

* Opencv fisheye distortion:

![OpenCVFisheyeDistortion](./images/OpenCVFisheyeDistortion.png)

Using `INSOffsetParser` to get the `INSOffsetParameter` internal parameters

```objc
INSImageInfoParser *parser = [[INSImageInfoParser alloc] initWithURL:urls.firstObject];
if ([parser open]) {
    INSExtraInfo *extraInfo = parser.extraInfo;
    
    INSOffsetParser *offsetParser =
    [[INSOffsetParser alloc] initWithOffset:parser.offset width:extraInfo.metadata.dimension.width
                                     height:extraInfo.metadata.dimension.height];
    for (INSOffsetParameter *param in offsetParser.parameters) {
        NSLog(@"Internal parameters: %@", param);
    }
}
```

### try the sample project and have a look at the header files for more details
