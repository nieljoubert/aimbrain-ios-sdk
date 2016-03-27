#AimBrain SDK integration

## Application class
In order to integrate the AimBrain iOS SDK it is necessary to set up `AMBNCapturingApplication` (subclass of `UIApplication`) as main application. `main.m` file after modification should look like this:

```objective_c
@import UIKit;
#import "MyAppDelegate.h"
#import "AMBNCapturingApplication.h"

int main(int argc, char * argv[])
{
    @autoreleasepool {
        return UIApplicationMain(argc, argv, NSStringFromClass([AMBNCapturingApplication class]), NSStringFromClass([MyAppDelegate class]));
    }
}
```

## API authentication
In order to communicate with the server, the application must be configured with a valid API Key and secret. Relevant configuration parameters should be passed to the SDK using the `AMBNManager`’s `configureWithApplicationId:secret` method. Most often the best place for it is  `application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions` of app delegate.

```objective_c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    [[AMBNManager sharedInstance] configureWithApplicationId:@"test" secret:@"secret"];
    return YES;
}
```

# Behavioural module

## Registering views
The more views have identifiers assigned, the more accurate the analysis can be made. Views can be registered using `registerView:withId` method of `AMBNManager`

```objective_c
- (void)viewDidLoad {
    [super viewDidLoad];

    [[AMBNManager sharedInstance] registerView:self.view withId:@"sign-in-vc"];
    [[AMBNManager sharedInstance] registerView:self.emailTextField withId:@"email-text-field"];
    [[AMBNManager sharedInstance] registerView:self.pinTextField withId:@"pin-text-field"];
    [[AMBNManager sharedInstance] registerView:self.wrongPINLabel withId:@"pin-label"];
}
```



## Starting collection
In order to start collecting behavioural data `start` method needs to be called on `AMBNManager`

```objective_c
[[AMBNManager sharedInstance] start];
```


## Session
In order to submit behavioural data to AimBrain `AMBNManager` needs to be configured with a session. There are two ways of doing it.

### Obtaining new session
A new session can be obtained by passing `userId` to `createSession` method on `AMBNManager`.

```objective_c
[[AMBNManager sharedInstance] createSessionWithUserId:userId completion:^(NSString *session, NSError *error) {
    if(session){
        //Do something after successful session creation
    }
}];
```

The manager is automatically configured with the obtained session ID.

### Configuring with existing session
A session can be stored and later used to configure `AMBNManager`.

```objective_c
[AMBNManager sharedInstance].session = storedSession
```

## Submitting behavioural data
After the manager is configured with a session, behavioural data can be submitted.

```objective_c
-(void) submitBehaviouralData {
    [[AMBNManager sharedInstance] submitBehaviouralDataWithCompletion:^(AMBNResult *result, NSError *error) {
        NSNumber * score = result.score;
        //Do something with obtained score
    }];
}
```

Server responds to data submission with the current behavioural score and status.

### Periodic submission
In order to schedule periodic submission use the following snippet:

```objective_c
// Call this method after the session ID is obtained
- (void) startPeriodicUpdate {
   self.timer = [NSTimer scheduledTimerWithTimeInterval:10.0f target: self selector:@selector(submitBehaviouralData) userInfo:nil repeats:true];
}

-(void) submitBehaviouralData {
    [[AMBNManager sharedInstance] submitBehaviouralDataWithCompletion:^(AMBNResult *result, NSError *error) {
        NSNumber * score = result.score;
        //Do something with obtained score
    }];
}
```

## Getting the current score
To get the current session score from the server without sending any data use `getScoreWithCompletion` method from `AMBBManager`.

```objective_c
[[AMBNManager sharedInstance] getScoreWithCompletion:^(AMBNResult *result, NSError *error) {
  //Do something with the obtained score
}]
```

## Sensitive data protection
It is possible to protect sensitive data. There are two ways of doing it.

### Limiting
Sensitive view data capture limiting is achieved by calling the `addSensitiveViews:` method with an array of sensitive views. Events captured on these views do not contain absolute touch location and the view identifiers are salted using device specific random key.

### Disabling
Disabled capturing of a view means absolutely no data will be collected about user activity connected with the view. Disabling means creating a `PrivacyGuard` object. It is required to store a strong reference to this object. Garbage collection of the privacy guard means that capturing will be enabled again.


>Capturing can be disabled for selected views. Array of protected views must be passed to `disableCapturingForViews` method.

```objective_c
- (void)viewDidLoad {
    [super viewDidLoad];
    self.pinPrivacyGuard = [[AMBNManager sharedInstance] disableCapturingForViews:@[self.pinTextField]];
}
```


>Another option is to disable all views. Not data will be captured until `PrivacyGuard` is invalidated (or garbage collected)

```objective_c
- (void)viewDidLoad {
    [super viewDidLoad];
    self.allViewsGuard = [[AMBNManager sharedInstance] disableCapturingForAllViews];
}
```



>Capturing can be re-enabled by calling `invalidate` method in the `PrivacyGuard` instance

```objective_c
- (void)onPinAccepted {
    [super viewDidLoad];
    [self.pinPrivacyGuard invalidate];
}
```

# Facial module

## Taking pictures of the user's face
In order to take a picture of the user's face the `openFaceImagesCaptureWithTopHint` method has to be called from the `AMBNManager`. The camera view controller is then opened and completion block is called after user takes the picture and the view is dismissed.
```objective_c
[[AMBNManager sharedInstance] openFaceImagesCaptureWithTopHint:@"To authenticate please face the camera directly and press 'camera' button" bottomHint:@"Position your face fully within the outline with eyes between the lines." batchSize:3 delay:0.3 fromViewController:self completion:^(BOOL success, NSArray *images) {
...
}];
```

## Authenticating with the facial module
In order to authenticate with facial module, the `authenticateFaceImages` method has to be called from the `AMBNManager`. An array with the images of the face has to passed as a parameter. The completion block is called with the score returned by the server, the score being between 0.0 and 1.0 and a liveliness rating, indicating if the photos were taken of a live person.
```objective_c
[[AMBNManager sharedInstance] authenticateFaceImages:images completion:^(NSNumber *score, NSNumber *liveliness, NSError *error) {
...
}];
```

## Enrolling with the facial module
Enrolling with the facial module is done by calling the `enrollFaceImages` method from the `AMBNManager`. An array with with the images of the face has to passed as a parameter. The completion block is called after the operation if finished. `success` indicates if operation was successful, `imagesCount` indicates how many images were received, processed successfully and had a face in them.
```objective_c
[[AMBNManager sharedInstance] enrollFaceImages:images completion:^(BOOL success, NSNumber *imagesCount, NSError *error) {
...
}];
```

## Comparing faces
Two batches of images of faces can be compared using the `compareFaceImages` method of the `AMBNManager`. The method takes an array of images of the first face and an array of images of the second face (arrays contain one or more images). The completion block is called with the similarity score and the liveliness ratings of both faces.
```objective_c
[[AMBNManager sharedInstance] compareFaceImages:firstFaceImages toFaceImages:secondFaceImages completion:^(NSNumber *result, NSNumber *firstLiveliness, NSNumber *secondLiveliness, NSError *error) {
...
}];
```
