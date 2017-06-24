# Disa Messaging

## New Message and New Message Extended
A plugin will will implement the `Disa.Framework.INewMessage` interface to participate in new message creation. `INewMessage` defines the following API:

| API | Description |
| --- | --- |
| GetContacts | |
| GetContactsFavorites | |
| GetContactPhoto | |
| FetchBubbleGroup | |
| FetchBubbleGroupAddress | |
| GetContactFromAddress | |
| MaximumParticipants | |
| FastSearch | |
| CanAddContact | | 
 
 In addition, a Plugin can implement the `Disa.Framework.INewMessageExtended` interface to provide additional capabilities in new message creation. `INewMessageExtended` defines the following API:

 | API | Description |
 | --- | --- |
 | FetchBubbleGroupAddressFromLink | |
 | SupportsShareLinks | |
