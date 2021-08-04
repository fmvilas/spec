## TL;DR: What's new?

* **Remotes**: A remote is a remote server our application will connect to send and receive messages. Most of the times, this is a broker but can also be a WebSocket or HTTP Server-Sent Events server.
* **Channel Identifiers**: Channels are not indentified by their "address" anymore. Instead, we define an ID that represents the channel and the "address" is specified separately. This encourages the reuse of channels and eases the process of referencing them and changing their address without breaking the referencing documents.
* **Operations**: There's a new top-level object called "operations". It defines the operations (as in actions) this application is performing or going to perform, and in which channel.
* **Send/Receive**: We get rid of "publish" and "subscribe" and instead replace them with "send" and "receive". They're both actions an application performs instead of actions an application expects the client to perform. Aside from the confusion "publish" and "subscribe" is creating, some people are incorrectly assuming that AsyncAPI is only meant to describe pub/sub event-driven architectures.
* **Optional channels**: from now on, the `channels` object is optional. The only two fields that are required are `asyncapi` and `info`.
* **More reusability**: now `servers`, `remotes`, and `channels` can be defined inside `components` for reusability purposes. This way, an AsyncAPI document may only contain a `components` object that can serve as an organization-wide menu of servers, remotes, channels, messages, etc.

## Abstract

For some years now, we've been discussing how to solve the publish/subscribe perspective problem. Some were arguing we should be giving priority to broker-based architectures because "they are the most common type of architecture AsyncAPI is used for". Although they're not wrong, by simply changing the meaning of `publish` and `subscribe` keywords, we'd be leaving many people and use cases out. We have to come up with a way to fix this problem without losing the ability to define client/server interactions too, which are especially useful for public WebSockets and HTTP Server-Sent Events APIs. And more importantly, we need a structure that will allow us to grow well and [meet our vision](https://www.asyncapi.com/roadmap).

## Foundational concepts

We gotta start from the beginning: an AsyncAPI file represents an application. This doesn't change. However, if no `channels` or `operations` are provided, the file becomes a "menu", a "library", a collection of reusable items we can later use from other AsyncAPI files.

We have been very focused in the meaning of `publish` and `subscribe` but there is another key aspect of the spec that is confusing to many people: `servers`. As it is right now, the `servers` keyword holds an object with definitions of the servers (brokers) we may have to connect to. However, it also holds information about the server our application is exposing (HTTP server, WebSocket server, etc.) In some tools, we have been assuming that if someone specifies `ws` as the protocol, it means they want to create a WebSocket server instead of a WebSocket client. But what if someone wants to connect to a broker using the WebSocket protocol? This whole thing about the role of our application has been confusing all of us. As @lornajane pointed out in multiple occassions, an application can be both a server and a client, or just a server, or just a client. Therefore, `servers` can't be made up of that mix. Exposed server interfaces and remote servers (usually brokers) have to be separated because —even though they look the same— they're semantically different.

## Remotes vs Servers

This proposal introduces the concept of `remotes`. Remotes are "remote servers" our application has to connect to. They're usually brokers but can also be other kind of servers. The name `remote` is taken from the Git specification, which references a remote repository.

On the other hand, `servers` are server interfaces our application expose. Their URL define where clients can reach them.

#### Example

```yaml
asyncapi: 3.0.0

servers:
  test:
    url: ws://test.mycompany.com/ws
    protocol: ws
    description: The application creates a WebSocket server and listens for messages. Clients can connect on the given URL.

remotes:
  mosquitto:
    url: mqtt://test.mosquitto.org
    protocol: mqtt
    description: The application is connecting to the Mosquitto Test broker.
```

## New `channels` and `operations` objects

Another common concern related to current state of `publish` and `subscribe` is the channel reusability problem we encounter because these verbs are part of the channel definition. To avoid this problem, we remove the operation verbs from the channel definitions and move them to their own root object `operations`. Let's have a look at an example:

```yaml
asyncapi: 3.0.0

channels:
  userSignedUp:
    address: user/signedup
    message:
      payload:
        type: object
        properties:
          email:
            type: string
            format: email

operations:
  sendUserSignedUpEvent:
    channel: userSignedUp
    action: send
```

There are a few new things here:

* Channel `address`: it's the logical address where you can find this channel. Usually, this is the topic name, the routing key, the URL path, etc.
  * **So what's this `userSignedUp` then?** This is the channel identifier. It's an identifier that serves to reference the channel from another part of the document or another document.
  * **Why not using the address directly?** We use JSON Pointer in the $ref construct. Referencing `user/signedup` using JSON Pointer would be `user~1signedup` as opposed to `user/signedup` like many people would think. That's highly unreadable and error-prone, therefore I'm introducing channel identifiers.
  * Channel identifiers also facilitate the task of changing the address in a single place. For instance, if a topic name changes, it has to be changed in only one place and all the other files referencing this channel definition will be automatically updated.
* Operations object: it's the place where all the operations of this application are defined. `channel` hints against which channel is this operation performed and `action` is the type of operation, i.e., we're "sending" or "receiving".
  * **What's this `sendUserSignedUpEvent`**? It's the operation identifier. Yes, the old `operationId` is now mandatory and it's implicit, i.e., the `operationId` field doesn't exist anymore.
  * **Why isn't `channel` a `$ref` or JSON Pointer?** To keep things simple. If you're referring to a channel here, it must be defined in the `channels` object. If we allow `$ref` here, it means it can be dereferenced and therefore the channel ID would be lost. To make things easier and more consistent, this field is simply a string with the name of the channel ID.
  * **Why `send` and `receive`?** No, it's not that I'm hating `publish` and `subscribe` already :smile: I'm using `send` and `receive` here to avoid confusions for those thinking that AsyncAPI is only meant to describe pub/sub architectures. I think "send" and "receive" are pretty common verbs that shouldn't be linked to any super-very-special meaning.

## Organization and reusability at its best

I'm adding a few new objects to the `components` object: `servers`, `remotes`, and `channels`. Some may be already wondering "**why? don't we have `servers`, `remotes`, and `channels` already in the root of the document?**". To understand this decision, I think it's better if I just describe the reusability model I have in mind.

### Reusability

Some people expressed their interest in having a "menu" of channels, messages, servers, etc. They're not really interested in a specific application. In other words, they want an organization-wide "library". This is the `components` object. It's now possible to do something like the following:

```yaml
asyncapi: 3.0.0

info:
  title: Organization-wide definitions
  version: 3.4.22

components:
  servers:
    ...
  remotes:
    ...
  channels:
    ...
  messages:
    ...
```

This is now a valid AsyncAPI file and it can be referenced from other AsyncAPI files:

```yaml
asyncapi: 3.0.0

info:
  title: My MQTT client
  version: 3.1.9

remotes:
  mosquitto:
    $ref: 'common.asyncapi.yaml#/components/remotes/mosquitto'

channels:
  userSignedUp:
    $ref: 'common.asyncapi.yaml#/components/channels/userSignedUp'

operations:
  sendUserSignedUpEvent:
    channel: userSignedUp
    action: send
```

As you can see, I'm not defining any channel or remote in this file but instead I'm pointing to their definitions in the org-wide document (`common.asyncapi.yaml`). And this leads me to the other part of this section: "organization".

### Organization

The example above shows how we explicitly reference the resources (remotes, servers, and channels) that our application is using. Having them inside `components` doesn't mean they are making any use of it. And that's key. The file is split in two "sections": application-specific and reusable items:

```yaml
asyncapi: 3.0.0

##### Application-specific #####
info: ...
servers: ...
remotes: ...
channels: ...
operations: ...
##### Reusable items ######
components:
  servers: ...
  remotes: ...
  channels: ...
  schemas: ...
  messages: ...
  securitySchemes: ...
  parameters: ...
  correlationIds: ...
  operationTraits: ...
  messageTraits: ...
  serverBindings: ...
  channelBindings: ...
  operationBindings: ...
  messageBindings: ...
###########
```

So for those of you who were wondering before "**why? don't we have `servers`, `remotes`, and `channels` already in the root of the document?**": This is the reason, it's no different than it's right now in v2.x but I thought I'd make it more clear this time. Just because we put something in `components` doesn't mean the application is making any use of it.

## Life is better with examples

Say we have a social network, a very basic one. We want to have a website, a backend WebSocket server that sends and receives events for the UI to update in real-time, a message broker, and some other services subscribed to some topics in the broker.

We'll define everything that's common to some or all the applications in a file called `common.asyncapi.yaml`. Then, each application is going to be defined in its own AsyncAPI file, following the template `{app-name}.asyncapi.yaml`.

<details>
<summary>The <code>common.asyncapi.yaml</code> file</summary>

```yaml
asyncapi: 3.0.0

info:
  title: Organization-wide stuff
  version: 0.1.0

components:
  servers:
    websiteWebSocketServer:
      url: ws://mycompany.com/ws
      protocol: ws

  remotes:
    mosquitto:
      host: test.mosquitto.org
      protocol: mqtt

  channels:
    commentLiked:
      address: comment/liked
      remotes:
        - mosquitto
      message:
        ...
    likeComment:
      address: likeComment
      servers:
        - websiteWebSocketServer
      message:
        ...
    commentLikesCountChanged:
      address: comment/{commentId}/changed
      servers:
        - mosquitto
      message:
        ...
    updateCommentLikes:
      address: updateCommentLikes
      servers:
        - websiteWebSocketServer
      message:
        ...
```

</details>

<details>
<summary>The <code>backend.asyncapi.yaml</code> file</summary>

```yaml
asyncapi: 3.0.0

info:
  title: Website Backend
  version: 1.0.0

servers:
  websiteWebSocketServer:
    $ref: 'common.asyncapi.yaml#/components/servers/websiteWebSocketServer'

remotes:
  mosquitto:
    $ref: 'common.asyncapi.yaml#/components/servers/mosquitto'

channels:
  commentLiked:
    $ref: 'common.asyncapi.yaml#/components/channels/commentLiked'
  likeComment:
    $ref: 'common.asyncapi.yaml#/components/channels/likeComment'
  commentLikesCountChanged:
    $ref: 'common.asyncapi.yaml#/components/channels/commentLikesCountChanged'
  updateCommentLikes:
    $ref: 'common.asyncapi.yaml#/components/channels/updateCommentLikes'

operations:
  onCommentLike:
    action: receive
    channel: likeComment
    description: When a comment like is received from the frontend.
  onCommentLikesCountChange:
    action: receive
    channel: commentLikesCountChanged
    description: When an event from the broker arrives telling us to update the comment likes count on the frontend.
  sendCommentLikesUpdate:
    action: send
    channel: updateCommentLikes
    description: Update comment likes count in the frontend.
  sendCommentLiked:
    action: send
    channel: commentLiked
    description: Notify all the services that a comment has been liked.
```

</details>

<details>
<summary>The <code>frontend.asyncapi.yaml</code> file</summary>

```yaml
asyncapi: 3.0.0

info:
  title: Website WebSocket Client
  version: 1.0.0

remotes:
  websiteWebSocketServer:
    $ref: 'common.asyncapi.yaml#/components/servers/websiteWebSocketServer'

channels:
  likeComment:
    $ref: 'common.asyncapi.yaml#/components/channels/likeComment'
  updateCommentLikes:
    $ref: 'common.asyncapi.yaml#/components/channels/updateCommentLikes'

operations:
  sendCommentLike:
    action: send
    channel: likeComment
    description: Notify the backend that a comment has been liked.
  onCommentLikesUpdate:
    action: receive
    channel: updateCommentLikes
    description: Update the UI when the comment likes count is updated.
```

</details>

<details>
<summary>The <code>notifications-service.asyncapi.yaml</code> file</summary>

```yaml
asyncapi: 3.0.0

info:
  title: Notifications Service
  version: 1.0.0

remotes:
  mosquitto:
    $ref: 'common.asyncapi.yaml#/components/remotes/mosquitto'

channels:
  commentLiked:
    $ref: 'common.asyncapi.yaml#/components/channels/commentLiked'

operations:
  onCommentLiked:
    action: receive
    channel: commentLiked
    description: Sends a notification to the author when one of its comments is liked.
```

</details>


<details>
<summary>The <code>comments-service.asyncapi.yaml</code> file</summary>

```yaml
asyncapi: 3.0.0

info:
  title: Comments Service
  version: 1.0.0
  description: This service is in charge of processing all the events related to comments.

remotes:
  mosquitto:
    $ref: 'common.asyncapi.yaml#/components/remotes/mosquitto'

channels:
  commentLiked:
    $ref: 'common.asyncapi.yaml#/components/channels/commentLiked'
  updateCommentLikes:
    $ref: 'common.asyncapi.yaml#/components/channels/updateCommentLikes'

operations:
  onCommentLiked:
    action: receive
    channel: commentLiked
    description: Updates the likes count in the database and sends the new count to the broker.
  sendCommentLikesUpdate:
    action: send
    channel: commentLikesCountChanged
    description: Sends the new count to the broker after it has been updated in the database.
```

</details>