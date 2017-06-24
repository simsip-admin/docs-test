# Telegram Contacts

TelegramBotContact in Telegram contacts.

Telegram.FetchContacts has call 

    using (var client = new FullClientDisposable(this))
    {
        var response = (ContactsContacts)await client.Client.Methods.ContactsGetContactsAsync(
            new ContactsGetContactsArgs
            {
                Hash = string.Empty
            });
        contactsCache.AddRange(response.Users.OfType<User>().ToList());
        _dialogs.AddUsers(response.Users);
        return contactsCache;
    }

Telegram.GetContacts (INewMessage)

    using (var client = new FullClientDisposable(this))
    {
        var searchResult =
            TelegramUtils.RunSynchronously(
                client.Client.Methods.ContactsSearchAsync(new ContactsSearchArgs
                {
                    Q = query,
                    Limit = 50 //like the official client
                }));
        var contactsFound = searchResult as ContactsFound;
        var globalContacts = GetGlobalContacts(contactsFound);
        localContacts.AddRange(globalContacts);
    }

NewMessage.GetGlobalContacts had the filter for Bot removal
