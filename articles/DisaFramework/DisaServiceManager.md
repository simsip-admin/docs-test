# Disa Service Manager

`Disa.Framework.ServiceManager` controls the registration and lifecycle of your Plugin.


## Supporting Classes

`ServiceManager` depends on several supporting classes that you'll want to have a good understanding of before we dig deeper into the core of `ServiceManager`.

__ServiceBinding__

`ServiceManager` maintains a static `List` of `ServiceBinding` instances. This allows ServiceManager to track lifecycle state for a particular `Service`.

    {
        public Service Service { get; private set; }
        public ServiceFlags Flags { get; private set; }

        public ServiceBinding(Service service, ServiceFlags flags)
        {
            Service = service;
            Flags = flags;
        }
    }

__ServiceFlags__

The lifecycle state of a particular `Service` is represented by an instance of `ServiceFlags`. 

    private class ServiceFlags
    {
        public bool Running { get; set; }
        public bool Starting { get; set; }
        public bool ManualSettingsNeeded { get; set; }
        public bool ConnectionFailed { get; set; }
        public bool AuthenticationFailed { get; set; }
        public bool DisconnectionFailed { get; set; }
        public bool DeauthenticationFailed { get; set; }
        public bool Aborted { get; set; }
        public bool AbortedSpecial { get; set; }
        public bool Expired { get; set; }
    }

This allows `ServiceManager` to expose methods for querying the state of a particular `Service` such as:

    private static IEnumerable<Service> BindingQuery(Func<ServiceBinding, bool> predicate)
    {
        return from serviceBinding in ServicesBindings
                where predicate(serviceBinding)
                select serviceBinding.Service;
    }

    public static IEnumerable<Service> Running
    {
        get { return BindingQuery(binding => binding.Flags.Running); }
    }

    public static bool IsRunning(Service service)
    {
        return Running.FirstOrDefault(s => s == service) != null;
    }

## Querying for Services

Here is a summary of the `ServiceManager` APIs you can use to query the services based on their lifecycle state. 

| API | Description |
| --- | --- |
| Registered |  |
| RegisteredNoUnified |  |
| Starting |  |
| Running |  |
| RunningNoUnified |  |
| ManualSettingsNeeded |  |
| ConnectionFailed |  |
| AuthenticationFailed |  |
| DisconnectionFailed |  |
| DeauthenticationFailed |  |
| Expired |  |
| Aborted |  |
| AbortedSpecial |  |
| GetNonRegistered |  |
| GetRegistered |  |
| IsRegistered |  |
| IsStarting |  |
| IsRunning |  |
| IsExpired |  |
| IsAborted |  |
| IsAbortedSpecial |  |
| IsManualSettingsNeeded |  |
| IsConnectionFailed |  |
| IsAuthenticationFailed |  |
| IsDisconnectionFailed |  |
| IsDeauthenticationFailed |  |
| GetFlags |  |

And here are some additional `ServiceManager` APIs for querying for `Service`s and `Service` state.

| API | Description |
| --- | --- |
| Get(BubbleGroup group) |  |
| Has(BubbleGroup group, Service service) |  |
| Get<T>(string guid) |  |

## Registering and Unregistering Services

Now we can return to `PlatformManager.InitializeMain` and the call to:

    ServiceManager.Initialize(allServices.ToList());

Recall that the collection of `Service` instance we have here was passed in to `InitializeMain`.  Typically, this will be called by the Disa front-end. We will see how this list of `Service` instances is created in the section on Deploying. For now, let's pickup in `ServiceManager.Initialize` where see that we assign the passed in `Service` instances to the collection `AllInternal`.

    AllInternal = allServices;

While `AllInternal` is private, it is exposed via the following APIs:

| API | Description |
| --- | --- |
| All |  |
| AllNoUnified |  |
| Get(Type serviceType) |  |
| GetByName(string serviceName) |  |
| GetUnified() |  |

We then get our first exposure to `ServiceManager.RegisteredServicesDatabase` via this call:

    RegisteredServicesDatabase.RegisterAllRegistered();

**ServiceManager.RegisteredServicesDatabase**

`RegisteredServicesDatabase` maintains an XML listing of all registered services. The XML backing file is named `RegisteredServicesList.xml` and will be located at the location pointed to by `Platform.GetSettingsPath()`. Besides  `RegisterAllRegistered`, which we will explore in a sec, `RegisteredServicesDatabase also exposes:

| API | Description |
| --- | --- |
| SaveAllRegistered | Saves the name of each `Service` in `RegisterNoUnified` into `RegisteredServicesList.xml` |
| FetchAllRegistered(string settingsPath) | Given a settings path, will return a `List<string>` of all registered service names. |
| AddToRegisteredAndSaveAllForImminentRestart(string settingsPath, string additionalService) | Given a settings path and the name of a new `Service`, will write out a new `RegisteredServicesList.xml` with new `Service` name included.

OK, let's get back to our `RegisteredServicesDatabase.RegisterAllRegistered` call. In this method, we read in all the `Service` names in `RegisteredServicesList.xml`. We then loop over all the names and verify that it is contained in `AllInternal`. With this check in place we proceed to call:

    Register(service);

Following into this function, we see that a `Service` is considered registered once a `ServiceBinding` instance has been added into the `ServiceManager.ServiceBinding` collection for it. Also, the `ServiceBinding` will have freshly initialized `ServiceFlags` instance at this point in time.

    lock (ServicesBindings) ServicesBindings.Add(
        new ServiceBinding(service, new ServiceFlags()));

**Unregistering**

A `Service` can be unregistered by calling `Unregister`. This will have the effect of removing the `ServiceBinding` instance for the `Service`. This will also cause an event to be raised that you can listen for:

    ServiceEvents.RaiseServiceUnRegistered(service);

**TODO**
    SettingsChangedManager.SetNeedsContactSync(service, true);

## Managing Service Lifecycle

**Starting**

`ServiceManager.Start` is used to start a `Service`. We start by wrapping the entire start of the `Service` in a background thread:

    return Task.Factory.StartNew(() =>
    {

We then further wrap the entire start of the `Service` in the DisaStart `WakeLock`.

    using (var wakeLock = Platform.AquireWakeLock("DisaStart"))
    {

Simplistic checks are then performed to make sure the `Service` is not already running or starting. We simply return if so. We then add one more additional wrapping of the start of the `Service` by locking on the service instance passed in:

    lock (service)
    {

With this setup in place, we now set our current state for this `Service`:

    GetFlags(service).Aborted = false;
    GetFlags(service).AbortedSpecial = false;
    GetFlags(service).Starting = true;
    GetFlags(service).ManualSettingsNeeded = false;

We then attempt to load our `DisaSettings` derived settings class for this `Service`.

    var settings = SettingsManager.Load(service);

If this returns null, it means we have not specified a `DisaSettings` derived class for this `Service` - see the section on Disa Settings in this Guide for further details on why this could occur. If the settings are null, then we call our `Service` lifecycle method `InitializeDefault`.

    if (!service.InitializeDefault())
    {
        GetFlags(service).ManualSettingsNeeded = true;
        ServiceEvents.RaiseServiceManualSettingsNeeded(service);
    }
    else
    {
        Utils.DebugPrint("Service initialized under no settings.");
    }

Note here how if our implementation for `Service.InitializeDefault` returns false, then we set our `ManaulSettingsNeeded` flag and raise the event `ManualSettingsNeeded`.

Now let's take the opposite path where are our call to SettingsManager.Load returns our DisaSettings derived settings class instance. In this case, we call the `Service` lifecycle method `Initialize` passing in our settings instance.

    if (service.Initialize(settings))
    {
        Utils.DebugPrint("Successfully initialized service!");
    }
    else
    {
        GetFlags(service).ManualSettingsNeeded = true;
        ServiceEvents.RaiseServiceManualSettingsNeeded(service);
    }

Similar to our implementation for `InitializeDefault`, if our implementation for `Initialize` returns false, then we set our `ManaulSettingsNeeded` flag and raise the event `ManualSettingsNeeded`.

Next, if our `Service` has specified that it `UsesInternet` we perform our platform specific checks to see if we have an Internet connection and if we should attempt an Internet connection. The `ServiceInfo` attribute on your Plugin's `Service` derived class will be where you specify your `Service` uses Internet.

With these initial checks and initialization lifecycle calls out of the way we now call `StartInternal` which will handle our connect and authenticate lifecycle method calls.

    StartInternal(service, wakeLock);

Picking up in `StartInternal`, we make our checks to see if manual settings are needed, the service is registered and the service is not already running. An appropriate `ServiceSchedulerException` is thrown if necessary. We then update our state for the `Service` with a call to `ClearFailures` which will update the state as follows:

    flags.ConnectionFailed = false;
    flags.AuthenticationFailed = false;
    flags.DeauthenticationFailed = false;
    flags.DisconnectionFailed = false;

Following this, we raise the `SettingsLoaded` event for any interested listeners.

With all of this behind us, we are now ready to call our lifecycle methods for connect and authenticate. A `Service` can specify via the `ServiceInfo` attribute the order in which these lifecycle methods are called as we can see here:

    if (registeredService.Information.Procedure 
        == ServiceInfo.ProcedureType.AuthenticateConnect)
    {
        authenticate();
        connect();
    }
    else if (registeredService.Information.Procedure 
                == ServiceInfo.ProcedureType.ConnectAuthenticate)
    {
        connect();
        authenticate();
    }

Note that the `connect` and `authenticate` calls here are actually `Action` delegates defined above this code snippet. The delegates handle catching and rethrowing exceptions as appropriate. Also, the delegates will handle setting the `ConnectionFailed` and `AuthenticationFailed` states for the `Service` if necessary.

If the calls to `connect` and `authenticate` succeed, then we set the state of the `Service` to `Running` and raise the `Started` event.

OK, back in our `ServiceManager.Start` method, we pickup by handling any exceptions coming out of our most recent steps. Exception handlers here will set the `Starting` state of the `Service` to false. In addition the `ServiceSpecialRestartException` handler will attempt another start of the `Service` with an appropriate delay:

    catch (ServiceSpecialRestartException ex)
    {
        Utils.DebugPrint("Service " + service.Information.ServiceName +
                            " is asking to be restarted on connect/authenticate. This should be called sparingly, Disa can easily " +
            "break under these circumstances. Reason: " + ex + ". Restarting...");
        StopInternal(service);
        epilogue();
        Start(service, smartStart, smartStartSeconds);
        return;
    }

Also, the `ServiceExpiredException` handler will set the `Aborted` and `Expired` state for the `Service` as well as raise the `ServiceExpired` event.

    catch (ServiceExpiredException ex)
    {
        Utils.DebugPrint("The service " + service.Information.ServiceName +
                            " has expired: " + ex);
        GetFlags(service).Aborted = true;
        GetFlags(service).Expired = true;
        ServiceEvents.RaiseServiceExpired(service);
        StopInternal(service);
        epilogue();
        return;
    }

A this point we hook into the `BubbleManager` and start our Bubble receiving thread.

    BubbleManager.SendSubscribe(service, true);
    BubbleManager.SendLastPresence(service);

    service.ReceivingBubblesThread = new Thread(() =>
    {
        StartReceiveBubbles(service);
    });
    service.ReceivingBubblesThread.Start();


We'll hold off on describing these for the section on the Bubble Manager and Bubbles later in this Guide.

Finally, we set the `Starting` state for our `Service` to false.

    GetFlags(service).Starting = false;

**TODO**

    BubbleQueueManager.SetNotQueuedToFailures(service);

    Utils.Delay(1000).ContinueWith(x =>
    {
        BubbleGroupSync.ResetSyncsIfHasAgent(service);
        BubbleGroupUpdater.Update(service);
        BubbleQueueManager.Send(new[] {service.Information.ServiceName});
        BubbleGroupManager.ProcessUpdateLastOnlineQueue(service);
        SettingsChangedManager.SyncContactsIfNeeded(service);
    });






















**Stopping**

**Restarting**

**Aborting**





