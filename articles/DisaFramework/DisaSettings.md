# Disa Settings


The Disa Framework provides four categories of settings that you should be familiar with to support your Plugin's needs.

| Category | Description |
| --- | --- |
| `DisaSettings` | A settings store of your service. You can use this to store any information. |
| `DisaMutableSettings` |  Use `DisaMutableSettings` and `MutableSettingsManager` to save information you find yourself frequently saving (such as a timestamp you need to keep updated everytime the service is started).
 |
| PluginSettingsUI/IPluginPage | Plugin defined Xamarin.Forms page or pages to present a UI to capture settings from the user |
| `DisaServiceUserSettings` | User settings such as `Ringtone` and `VibrateOption` |

## DisaSettings

**Setup and Initialization**

Typically you will derive a class from `DisaSettings` to hold your Plugin specific settings. While we haven't discussed implementing your Plugin `Service` class yet, we can briefly mention the touch-points for your `DisaSettings`. The `ServiceInfo` attribute on your `Service` class takes a `Type` parameter of your `DisaSettings` derived class as can be seen here:

    [ServiceInfo("WackyMessenger", true, false, false, false, false, typeof(WackyMessengerSettings), 
            ServiceInfo.ProcedureType.ConnectAuthenticate, typeof(TextBubble))]

This will have the effect of storing away a `ServiceName` ("WackyMessenger") and a  `Type` (typeof(WackyMessengerSettings)) for your `DisaSettings` derived class at:

    Service.Information.ServiceName

and

    Service.Information.Settings

We'll see how this is used shortly. Now when your Plugin's `Service` derived class instance is started, the first thing that happens is that your override of `InitializeDefault()` is called. It attempts to try and start the service without any settings. If this method returns true, it is assumed your service doesn't need any settings. If this method returns false, then `InitializeDefault(DisaSettings)` is called - the framework will provide you with your stored settings at this point.

**Managing**

`Disa.Framework.SettingsManager` manages saving, loading and deleting your `DisaSettings` derived class instance. Whenever you want to save your settings, you can call `SettingsManager.Save`.

    public static void Save(Service service, DisaSettings settings)

This method will first determine a path for your settings based on the `Service.Information.ServiceName`:

    Path.Combine(Platform.GetSettingsPath(), service.Information.ServiceName + ".xml");

It will then hand-off completion of saving to:

    public static void Save(Stream fs, Type settingsType, DisaSettings settings)

And here we see how the `Type` you originally specified for your settings is used to serialize an XML representation of your settings:

    var serializerObj = new XmlSerializer(settingsType);
    serializerObj.Serialize(fs, settings);

Since your settings are based off of your `ServiceName`, loading and deleting your settings have simple APIs:

    public static DisaSettings Load(Service ds)
    public static void Delete(Service service)

Note that if your settings have not been saved out yet, `Load` will return `null`.

Additional public API's are availalbe in SettingsManager for loading and saving that allow you to specify stream, settings path, etc.

  

## DisaMutableSettings

**Setup**

Typically you will derive a class from `DisaMutableSettings` to hold frequently changing settings.

**Managing**

`Disa.Framework.MutableSettingsManager` manages saving, loading and deleting your `DisaMutableSettings` derived class instance. Whenever you want to save your settings, you can call `MutableSettingsManager.Save`.

    public static void Save<T>(T settings) where T : DisaMutableSettings

This method will call out to 

    Save(typeof(T).Name, settings);

Following this through we see that your DisaMutableSettings derived class will be serialized out to XML at:

    Path.Combine(Platform.GetSettingsPath(), name + ".xml");

Where `name` here is the class name of your `DisaMutableSettings` derived class.

Since your settings are based off of your class name, loading and deleting your settings have simple APIs:

    public static T Load<T>() where T : DisaMutableSettings
    public static void Delete<T>() where T : DisaMutableSettings

Note, that in this case, `Load` will create an appropriate instance of your settings if a file backing has not been saved out yet:

    private static DisaMutableSettings Load(string name, 
                                            Type settings)
	{
        .
        .
        .
        return Activator.CreateInstance(settings)
            as DisaMutableSettings;
    }

## PluginSettingsUI/IPluginPage

Disa Plugins can present a Xamarin.Forms based UI to expose settings necessary for initializing and modifying attributes for their needs. An example would be to allow a user to input credentials to identify the user to a particular messaging back-end. 

To indicate to the Disa Framework a designated class for this, you annotate a class with the `Disa.Framework.PluginSettingsUI` attribute. You pass in the `Type` of your Plugin's `Service` derived class. Once you have identified a class with the `PluginSettingsUI` attribute, you can implement the `Disa.Framework.Mobile.IPluginPage` interface's `Fetch` method to return a Xamarin.Forms `Page` to present a UI to capture/modify settings particular to your Plugin.

## DisaServiceUserSettings

**Setup**

Recall in `PlatformManager.InitializeMain` the call to `ServiceUserSettingsManager.LoadAll`. If we dig into this we see that we loop over `ServiceManager`'s `AllNoUnified` collection of `Service`s. For each Plugin `Service` we call `ServiceUserSettingsManager.Load`. While this is a similar XML serialization/deserialization setup that we have seen above for the other settings, the important detail to note here is the determination of the settings path. The critical function to examine is listed here:

    private static string GetBaseLocation()
    {
        var databasePath = Platform.GetDatabasePath();
        if (!Directory.Exists(databasePath))
        {
            Utils.DebugPrint("Creating database directory.");
            Directory.CreateDirectory(databasePath);
        }

        var bubbleGroupsSettingsBasePath = Path.Combine(databasePath,
            "serviceusersettings");
        if (!Directory.Exists(bubbleGroupsSettingsBasePath))
        {
            Utils.DebugPrint(
                "Creating bubble service user settings base directory.");
            Directory.CreateDirectory(bubbleGroupsSettingsBasePath);
        }

        return bubbleGroupsSettingsBasePath;
    }

First, we determine, and create if necessary, the platform specific directory for our database location. Then, we append "serviceusersettings" and create, if necessary, this subdirectory. OK, with this location in hand, let's see how it is used. The critical function here is listed below:

    private static string GetPath(Service service)
    {
        return Path.Combine(GetBaseLocation(), 
            service.Information.ServiceName + ".xml");
    }

OK, we are familiar with this now. The `DisaUserSettings` for a particular Plugin is stored using the Plugin's `ServiceName`. For example, for the Telegram plugin, the path would be something like:

    <database path>\serviceusersettings\Telegram.xml

**Managing**

Currently, `DisaUserSettings` are typically managed by the Disa front-end. Here is a listing of the current settings maintained:

| Setting | Description |
| --- | --- |
| NotificationLed |  |
| Ringtone |  |
| BlockNotifications |  |
| ServiceColor |  |
| VibrateOption |  |
| VibrateOptionCustomPattern |  |
