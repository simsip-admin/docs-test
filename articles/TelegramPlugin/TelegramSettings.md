# Telegram Settings

Recall that the Disa Framework provides three categories of settings:

| Category | Description |
| --- | --- |
| DisaSettings | Settings tied to a particular Plugin |
| DisaMutableSettings | Miscellaneous settings based on your Plugin's needs |
| PluginSettingsUI/IPluginPage | Plugin defined Xamarin.Forms page or pages to present a UI to capture settings from the user |


`Disa.Framework.Telegram.Mobile.Settings` class handles settings for Telegram. To indicate to the Disa Framework we would like this class to present a Xamarin.Forms UI for settings, we annotate the class with the `PluginSettingsUI` attribute. We also implement the `Disa.Framework.Mobile.IPluginPage` interface's `Fetch` method to return a Xamarin.Forms `Page` to capture/modify settings for the Telegram plugin. In the `Fetch` implementation we see that we use the `ServiceManager.IsManualSettingsNeeded` to determine if we should display the UI for an initial setup or the UI for modifying already existing settings.

    if (ServiceManager.IsManualSettingsNeeded(service))
    {
        navigationPage = new NavigationPage(Setup.Fetch(service));
    }
    else
    {
        navigationPage = new NavigationPage(new Main(service));
    }
    return navigationPage;

`ManualSettingsNeeded` is a boolean flag on the `ServiceBindings` class which we will explore in more depth later.

Let's take the path for an initial setup first. Picking up in `Setup.Fetch` we see a setup for a Xamarin.Forms `TabbedPage` which will act as our setup wizard. At it's most simplest, the wizard will collect a phone number and then send an SMS code to the device to be sent back to Telegram. But let's get a more detailed overview in place just to provide proper context.

| Page | Description |
| --- | --- |
| Info | Collects phone number from user and displays a switch to allow user to load conversations. When Next is tapped, calls `Telegram.GenerateAuthentication` to authenticate the phone number and if the authentication is successful, moves to the next wizard page - `Code`.|
| Code | Coming from the `Info` page, we have a valid phone number to use. The Verify button will request a code be sent via SMS via a call to `Telegram.RequestCode`. The user will input the SMS code and then tap Submit. If the user has not input user info yet, the wizard will go to `UserInformation`, otherwise the wizard will call  `Telegram.RegisterCode`and then either navigate to `Password` or call `Setup.Save` to save the settings and end the wizard.  |
| Password | If the wizard has determined we need to enter a password then we are directed here to allow the user to enter a password which is verified by `Telegram.VerifyPassword`. This is followed by a call to `Setup.Save` and the wizard finishes. The "forgot password" button will take the user to the `PasswordCode` page. |
| PasswordCode |  We get here if the user has specified they forgot their password. A call to `Telegram.RequestAccountReset` is made from the constructor to setup the page. This will send an SMS code which can be entered on the page and a subsequent `Telegram.VerifyCode` will do the verification for us. If everything is ok then a call to `Setup.Save` is made and the wizard finishes. |
| UserInformation | For a first time registration we need to also collect first and last name. This page will collect that info and then call `Telegram.RegisterCode` followed by a call to `Setup.Save` and then the wizard will finish. |

Notice how there was a call to `Setup.Save` at all exit points of the wizard. Let's take a closer look at what is going on here.  The signature for this function is as follows:

    private static void Save(Service service, uint accountId, TelegramSettings settings)

  Since the Telegram plugin can represent different Telegram accounts keyed off of an appropriate Telegram phone number, the `TelegramSetupSettings` class that is derived from `DisaMutableSettings` records a collection of `TelegramSettings` keyed off of the Telegram phone number. The `TelegramSettings` class which is derived off of `DisaSettings` records the details for an individual Telegram phone number.

  Now back to `Setup.Save`. We see that we first save the current `TelegramSettings`:

    SettingsManager.Save(service, settings);

Then we start the actual plugin service:

    ServiceManager.Start(service, true);


