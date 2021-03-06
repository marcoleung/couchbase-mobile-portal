---
id: data-routing
title: Data routing
permalink: guides/sync-gateway/channels/index.html
---

Sync Gateway uses channels to make it easy to share a database between a large number of users and control access to the database. Channels are the intermediaries between documents and users. Every document in the database belongs to a set of channels, and every user is allowed to access a set of channels. You use channels to:

- Partition the data set.
- Authorize users to access documents.
- Minimize the amount of data synced down to devices.

## Introduction to channels

In Sync Gateway, a **channel** is like a combination of a tag and a message-queue. Channels have relationships to
both documents and users:

- Every document is associated with a set of channels. From the document's perspectives, the channels are like tags that can be used to identify its type, purpose, accessibility, etc. The app-defined sync function is responsible for assigning every incoming document revision a set of channels as it's saved to the database.

- Every user and role has a set of channels that they're allowed to read. A user can only read documents that are in at least one of the user's channels (or the channels of roles that user has.) User/role channel access can be assigned directly through the admin API, or via the sync function when a document is updated.

- A Couchbase Lite "pull" replication can optionally specify what channels it wants to receive documents from. (If it doesn't, it gets all channels the user has access to.) Client apps can use this ability to intelligently sync with a subset of the available documents from the database.

There's not much to a channel besides its name. Channels come into existence as documents are assigned to them. Channels with no documents assigned to them are empty.

Valid channel names consist of text letters `[A–Z, a–z]`, digits `[0–9]`, and a few special characters 	`[= + / . , _ @]`. Channel names are case-sensitive.

## Developing channels

Now that you've learned what a channel is as a concept, let's put it to practice and see how a channel definition is applied and how it affects the system.

### Mapping documents to channels

You assign documents to `channels` either by adding a channels property to the document or by using a sync function. No matter which option you choose, the channel assignment is implicit—the content of the document determines what channels it belongs to.

#### Using a channels property

Adding a `channels` property to each document is the easiest way to map documents to channels. The `channels` property is an array of strings that contains the names of the channels to which the document belongs. If you do not include a `channels` property in a document, the document does not appear in any channels.

#### Using a sync function

Creating a sync function is a more flexible way to map documents to channels. A sync function is a JavaScript function that takes a document body as input and, based on the document content, decides what channels to assign the document to. The sync function cannot reference any external state and must return the same results every time it's called on the same input.

You specify the sync function in the configuration file for the database. Each sync function applies to one database.

To add the current document to a channel, the sync function calls the special function `channel`, which takes one or more channel names (or arrays of channel names) as arguments. For convenience, `channel` ignores `null` or `undefined` argument values.

Defining a sync function overrides the default channel mapping mechanism (the document's `channels` property is ignored). The default mechanism is equivalent to the following simple sync function:

```javascript
function (doc) {
	channel (doc.channels);
}
```

Learn more about defining a sync function in our [Sync Function API guide](../sync-function-api-guide/index.html).

### Replicating channels to Couchbase Lite

If Couchbase Lite doesn't specify any channels to replicate, it gets all the channels to which its user account has access. Due to this behavior, most apps do not have to specify a channels filter—instead they can just do the default sync configuration on the specify the Sync Gateway database URL with no filter in order to replicate the channels of interest.

To replicate channels to Couchbase Lite, you configure the replication to use a filter named `sync_gateway/bychannel` with a filter parameter named `channels`. The value of the `channels` parameter is a comma-separated list of channels to fetch. The replication from Sync Gateway now pulls only documents tagged with those channels.

### Removing a replicated document

A document can be removed from a channel without being deleted. For example, this can happen when a new revision is not added to one or more channels that the previous revision was in. 

Subscribers (downstream databases pulling from this database) automatically handle the change. Sync Gateway's `_changes` feed includes one more revision of a document after it stops matching a channel. Couchbase Lite creates a special tombstone revision, adding `_removed:true` property to the entry when this happens.

This algorithm ensures that any views running in Couchbase Lite do not include an obsolete revision. The app code should use views to filter the results rather than just assuming that all documents in its local database are relevant.

If a user's access to a channel is revoked or Couchbase Lite stops syncing with a channel, documents that have already been synced are not removed from the user's device.

### Authorizing user access

The `all_channels` property of a user account determines which channels the user can access. Its value is derived from the union of:

- The user's `admin_channels` property, which is settable via the admin REST API.
[//]: # "TODO: Link to Programmatic Authorization section."
- The channels that user has been given access to by `access()` calls from sync functions invoked for current revisions of documents (see Programmatic Authorization).
- The `all_channels` properties of all roles the user belongs to, which are themselves computed according to the above 
two rules.

The only documents a user can access are those whose current revisions are assigned to one or more channels the user has access to:

- A GET request to a document not assigned to one or more of the user's available channels fails with a 403 error.
- The `_all_docs` property is filtered to return only documents that are visible to the user.
- The `_changes` property ignores requests (via the `channels` parameter) for channels not visible to the user.

Write protection — access control of document PUT or DELETE requests—is done by document validation. This is handled in the sync function rather than a separate validation function.

After a user is granted access to a new channel, the changes feed incorporates all existing documents in that channel, even those from earlier sequences than the current request's `since` parameter. That way the next pull request retrieves all documents to which the user now has access.

#### Programmatic Authorization

Documents can grant users access to channels. This is done by writing a sync function that recognizes such documents and calls a special `access()` function to grant access.

The `access()` function takes the following parameters: a user name or array of user names and a channel name or array of channel names. For convenience, null values are ignored (treated as empty arrays).

A typical example is a document that represents a shared resource (like a chat room or photo gallery). The document has a `members` property that lists the users who can access the resource. If the documents belonging to the resource are all tagged with a specific channel, then the following sync function can be used to detect the membership property and assign access to the users listed in it:

```javascript
function(doc) {
	if (doc.type == "chatroom") {
		access (doc.members, doc.channel_id)
	}
}
```

In the example, a chat room is represented by a document with a `type` property set to `chatroom`. The `channel_id` property names the associated channel (with which the actual chat messages are tagged) and the `members` property lists the users who have access.

The `access()` function can also operate on roles. If a user name string begins with `role:` then the remainder of the string is interpreted as a role name. There's no ambiguity here, because ":" is an illegal character in a user or role name.

Because anonymous requests are authenticated as the user "GUEST", you can make a channel and its documents public by calling `access` with a username of `GUEST`.

#### Authorizing Document Updates

Sync functions can also authorize document updates. A sync function can reject the document by throwing an exception:

```javascript
throw ({forbidden: "error message"})
```

A 403 Forbidden status and the given error string is returned to the client.

To validate a document you often need to know which user is changing it, and sometimes you need to compare the old and new revisions. To get access to the old revision, declare the sync function like this:

```javascript
function(doc, oldDoc) { ... }
```

`oldDoc` is the old revision of the document (or empty if this is a new document).

You can validate user privileges by using the helper functions: `requireUser`, `requireRole`, or `requireAccess`. Here's some examples of how you can use the helper functions:

```javascript
// throw an error if username is not "snej"
requireUser("snej")
// throw if username is not in the list
requireUser(["snej", "jchris", "tleyden"]) 
// throw an error unless the user has the "admin" role
requireRole("admin") 
// throw an error unless the user has one of those roles
requireRole(["admin", "old-timer"]) 
// throw an error unless the user has access to read the "events" channel
requireAccess("events") 
// throw an error unless the can read one of these channels
requireAccess(["events", "messages"]) 
```

Here's a simple sync function that validates whether the user is modifying a document in the old document's `owner` list:

```javascript
function (doc, oldDoc) {
	if (oldDoc) {
		requireUser(oldDoc.owner); // may throw({forbidden: "wrong user"})
	}
}
```

## Special channels

There are a two special channels that are automatically created when Sync Gateway starts:

- The **public channel**, written as `!` in the sync function. Documents added to this channel are visible to any user (i.e all users are automatically granted access to the `!` channel). This channel can be used as a public distribution channel.
- The **star** channel, written as `*` in the sync function. All documents are added to this channel. So any user that is granted access to the `*` channel can access all the documents in the database. A user can be given access to the **star** channel through the sync function or in the [configuration file](../config-properties/index.html#foo_user).
	- **Note 1:** Sync Gateway automatically assigns documents to the all docs channel. Explicitly assigning a document to it in the Sync Function (i.e `channel('*')`) will result in unexpected behavior such as receiving the document twice on the client side.
	- **Note 2:** The **star** channel doesn't mean that the user is granted access to all channels. It is only being granted access to 1 channel which contains **all documents**. This distinction is important when using the [requireAccess()](../sync-function-api-guide/index.html#requireaccesschannels) Sync Function method.

The following Sync Function maps the document to the public channel if it contains an `isPublic` property set to true and grants users with the 'admin' role access to the all docs channel.

```javascript
function (doc, oldDoc) {
	if (doc.isPublic) {
		channel('!');
	}
	if (doc.type == 'user') {
		requireRole('admin');
		access(doc.username, '*');
	}
}
```

## Troubleshooting channels

### Inspecting document channels

You can use the admin REST API to see the channels that documents are assigned to. Issue an `_all_docs` request, and add the query parameter `?channels=true` to the URL. Here's a command-line example that uses the [HTTPie](https://github.com/jkbrzt/httpie) tool (like a souped-up curl) to look at the channels of the document `foo`:

```bash
$ http POST 'http://localhost:4985/db/_all_docs?channels=true' keys:='["foo"]'

HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Length: 150
Content-Type: application/json
Date: Wed, 07 May 2014 21:52:17 GMT
Server: Couchbase Sync Gateway/1.00
{
    "rows": [
        {
            "id": "foo", 
            "key": "foo", 
            "value": {
                "channels": [
                    "short", 
                    "word"
                ], 
                "rev": "1-86effb929acbf953905dd0e3974f6051"
            }
        }
    ], 
    "total_rows": 16, 
    "update_seq": 26
}
```

The output shows that the document is in the channels `short` and `word`.

### Inspecting user/role channels

You can use the admin REST API to see what channels a user or role has access to:

```javascript
$ curl http://localhost:4985/db/_user/pupshaw

{
    "admin_channels": [
        "all"
    ],
    "admin_roles": [
        "froods"
    ],
    "all_channels": [
        "all",
        "hoopy"
    ],
    "name": "pupshaw",
    "roles": [
        "froods"
    ]
}
```

The output shows that the user `pupshaw` has access to channels `all` and `hoopy`.
