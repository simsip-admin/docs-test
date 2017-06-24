# Disa Platform Manager

The `PlatformManager` is responsible for exposing appropriate platform abstractions that a plugin can take advantage of. Take, for example, the common requirement for mobile developers to abstract the location for database files. Most mobile developers are familiar with writing the following platform specific logic for Android and iOS platforms:

**Android**

    string databasePath = System.Environment.GetFolderPath(
        System.Environment.SpecialFolder.Personal);

**iOS**

    string documentsPath = Environment.GetFolderPath (
        Environment.SpecialFolder.Personal);
    string databasePath = Path.Combine (
        documentsPath, "..", "Library");

While this is certainly good knowledge to have, a platform can have many more platform specific scenarios such as this one that would benefit from a common abstraction - allowing you to focus on your plugin logic instead. The `PlatformManager` and its supporting classes provide this benefit and in some cases enforce a certain behavior for a plugin to follow.

## PlatformImplementation

The `PlatformManager` manages classes derived from `PlatformImplementation`. It is in the derived class implementation that you'll find support for various  platform specific scenarios. `PlatformImplementation` is an abstract class with numerous abstract methods to build out a rich support for particular platform. Currently, Disa provides an Android and Desktop implementations. By familiarizing yourselft with `PlatformImplementation`'s API surface, you can take advantage of pre-built support for platform specific scenarios. Also, several of the APIs provide messaging and other plugin specific functionality that you will need to know to implement various plugin features. Here is a summary of the current API surface for `PlatformImplementation`:

| API | Description |
| --- | --- |
| MarkTemporaryFileForDeletion | |
| UnmarkTemporaryFileForDeletion |  |
| GetIcon |  |
| GetCurrentLocale |  |
| GetFilesPath |  |
| GetPicturesPath |  |
| GetVideosPath |  |
| GetAudioPath |  |
| GetLogsPath |  |
| GetSettingsPath |  |
| GetDatabasePath |  |
| GetDeviceId |  |
| GetPhoneBookContacts |  |
| ScheduleAction |  |
| RemoveAction |  |
| ScheduleAction |  |
| WakeLock |  |
| AquireWakeLock |  |
| OpenContact |  |
| DialContact |  |
| LaunchViewIntent |  |
| DeviceHasApp |  |
| HasInternetConnection |  |
| ShouldAttemptInternetConnection |  |
| GetMimeTypeFromPath |  |
| GetExtensionFromMimeType |  |
| GenerateJpegBytes |  |
| GenerateVideoThumbnail |  |
| GenerateBytesFromContactCard |  |
| GenerateContactCardFromBytes |  |
| GenerateLocationThumbnail |  |
| CreatePartyBitmap |  |
| GetCurrentBubbleGroupOnUI |  |
| SwitchCurrentBubbleGroupOnUI |  |
| DeleteBubbleGroup |  |
| ExecuteAllOldWakeLocksAndAllGracefulWakeLocksImmediately |  |


## Platform Initialization
Platform initialization is handled by the Disa client. `PlatformManager` exposes the following API to inject a particular `PlatformImplementation`:

    public static void PreInitialize(PlatformImplementation platform, 
                                     AxolotlImplementation axolotl)

`AxolotlImplementation` provides support for the Axolotl messaging protocol which provides perfect forward secrecy. We will discuss this in another section of the Guide. `PreInitialize` simply assigns the injected `PlatformImplementation` to:

    PlatformManager.PlatformImplementation

and

    Platform.PlatformImplementation

**TODO: Should we DRY this up?**

The assignment to `Platform.PlatformImplementation` is for convenience as `Platform` exposes a simpler API to call such as:

    Platform.GetDatabasePath();

With `PlatformManager.PreInitialize` completed, the Disa client will then call `PlatformManager.InitializeMain`.

    public static void InitializeMain(Service[] allServices) 

An array of `Service` instances is passed into `InitializeMain`. How this list is built up by the Disa client will be discussed in the Deploying section of this Guide. In `InitializeMain`, the `Service` instances are passed in to `ServiceManager.Initialize`. 

    ServiceManager.Initialize(allServices.ToList());

See the section on `ServiceManager` in this Guide for further details here. Then, a call is made to `ServiceUserSettingsManager.LoadAll`.

    ServiceUserSettingsManager.LoadAll();

See the section Settings in the Guide for further details. Finally, a call is made to `BubbleGroupFactory.LoadAllPartiallyIfPossible`.

    BubbleGroupFactory.LoadAllPartiallyIfPossible();

See the section on Bubbles and BubbleGroups for further details. OK, we punted on most of the description that is going on here, but it will make more sense when we pick up the thread in each of the sections devoted to each particular piece of functionality.

## Versioning

`PlatformManager' is the offical location to determine the version of the Disa.Framework you are coding against. The APIs `FrameworkVersion` and 'FrameworkVersionInt` are used for this.

## Assemblies

`PlatformManager` maintains an official list of core assemblies needed when deploying your Plugin. We will see how this list is used in the Deploying section.



