# Disa Contacts

`Disa.Framework.Contact` is an abstract class that Plugins will derive from for their own Plugin specific needs (e.g., TelegramContact).

`Contact` has the following fields:

| Field | Description |
| --- | --- |
| FirstName | |
| LastName | |
| Status | |
| Ids | |
| LastSeen | |
| Available | |
| FullName | |

And `Contact.ID` has the following fields:

| Field | Description |
| --- | --- |
| Service | |
| Id | |
| LegibleId | |
| Name | |
| Tag | |

Deriving from `Contact` are `PartyContact` and `BotContact`. Plugins will derive from these classes as necessary to indicate that a Contact is participating in a Party or that a Contact is a Bot.

