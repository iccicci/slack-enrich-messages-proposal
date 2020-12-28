# slack-enrich-messages-proposal

New feature proposal request for Slack

# Preface

At [TLM Partners](https://tlmpartners.com/) we are working on a Slack App to enrich the messages with useful information for us. One of the specific needs we have is the
ability to customize additional information according to the user who reads the message.

After several attempts we are still not satisfied with the visual rendering of our App and after having thoroughly studied the [Slack-Api](https://api.slack.com/) we believe
that it is not possible to achieve the result we want with the tools that Slack makes available to us today.

For this reason we make this request for a new feature.

# Scenario

Our App is installed in our workspace.

* `T00000001` - is our team (AKA: workspace)
* `B0000000001` - is the bot user of our App
* `C0000000001` - is a standard public channel in our workspace
* `U0000000001` - is a user, not necessarily one of us
* `U0000000002` - is one of us, the user we want to show the additional information, it authenticated through [Slack OAuth 2.0](https://api.slack.com/authentication/oauth-v2)
* `D0000000001` - is the direct message channel between `B0000000001` and `U0000000002`

* `{ "channel": "C0000000001", "ts": "1600000001.000000" }`

  is a standard message posted by `U0000000001`: the message we want to enrich, from now on the `target message`

* `{ "channel": "C0000000002", "ts": "1600000002.000000" }`

  is the message we need to be implemented, from now on the `enrich message` (**the core of this new feature proposal request**)

### Target message

This is the detailed `target message`; once `U0000000002`'s client receives this message it should render the message as usual.

```JSON
{
  "client_msg_id": "00000000-0000-0000-0000-000000000001",
  "type":          "message",
  "text":          "test message",
  "user":          "U0000000001",
  "ts":            "1600000001.000000",
  "team":          "T00000001",
  "channel":       "C0000000001",
  "event_ts":      "1600000001.000000",
  "channel_type":  "channel",
  "blocks":        [
    {
      "type":     "rich_text",
      "block_id": "blk0",
      "elements": [
        {
          "type":     "rich_text_section",
          "elements": [
            {
              "type": "text",
              "text": "test message"
            }
          ]
        }
      ]
    }
  ]
}
```

### Enrich message

This is the detailed `enrich message`; we have no specific needs about how it should be rendered by `U0000000002`'s client, for now let's say it can be rendered as usual
(details will follow). Please note the presence of the `enrich` key, it doesn't exist in Slack, it is the core of this proposal.

```JSON
{
  "client_msg_id": "00000000-0000-0000-0000-000000000002",
  "type":          "message",
  "text":          "",
  "user":          "B0000000001",
  "ts":            "1600000002.000000",
  "team":          "T00000001",
  "channel":       "D0000000001",
  "event_ts":      "1600000002.000000",
  "channel_type":  "im",
  "enrich":        [
    {
      "channel": "C0000000001",
      "ts":      "1600000001.000000",
      "blocks":  [
        {
          "type":     "divider",
          "block_id": "blk1"
        },
        {
          "type":     "context",
          "block_id": "blk2",
          "elements": [
            {
              "type": "plain_text",
              "text": "useful info"
            }
          ]
        }
      ]
    }
  ]
}
```

# The proposal

What we need is that `U0000000002`'s client (once received both `target message` and `enrich message`) starts rendering the `target message` with its own `blocks` plus
the ones added by the `enrich message`; i.e. as if it was:

```JSON
{
  "client_msg_id": "00000000-0000-0000-0000-000000000001",
  "type":          "message",
  "text":          "test message",
  "user":          "U0000000001",
  "ts":            "1600000001.000000",
  "team":          "T00000001",
  "channel":       "C0000000001",
  "event_ts":      "1600000001.000000",
  "channel_type":  "channel",
  "blocks":        [
    {
      "type":     "rich_text",
      "block_id": "blk0",
      "elements": [
        {
          "type":     "rich_text_section",
          "elements": [
            {
              "type": "text",
              "text": "test message"
            }
          ]
        }
      ]
    },
    {
      "type":     "divider",
      "block_id": "blk1"
    },
    {
      "type":     "context",
      "block_id": "blk2",
      "elements": [
        {
          "type": "plain_text",
          "text": "useful info"
        }
      ]
    }
  ]
}
```

More than this we obviously need a change to API too in order to send such messages.

# Options

### API

We have no specific requests about the API: the desired result could be achieved through many options.

It could be done through a simple change to [chat.postMessage](https://api.slack.com/methods/chat.postMessage) making it accept the new `enrich` key. This option would
also allow for easy to update the message through the [chat.update](https://api.slack.com/methods/chat.update) method, which would be an intgeresting nice to have.

It could be through a new dedicated API method (ex. `chat.postEnrich`).

It could be through any other idea with the lesser required effort.

### Number of enriched messages

In the `enrich message` example the `enrich` value is an array, it would mean that more than one message could be enriched; this could help to collect more enrichments
in only one message, but this is not strict; only one enrichment per message could be a helpful result anyway, if this could help reduce the required effort.

### Message type

In the `enrich message` example the `type` value is `message`; we don't know all the mechanics behind the message type, if adding a new dedicated message type could be
helpful to reduce the required effort, why not?

A new message type could be also helpful to instruct the clients about a dedicated rendering for the `enrich message`s.

### Rendering

In our mind not rendering at all the `enrich message` could be fine, but we guess that it could be the source of (at least) a problem. When a client starts it loads only
the last messages of the channels and previous messages (from newer to older) are loaded while the user scrolls up: making the `enrich message` to have a vertical size
could help handling stuff like this.

Any simple static rendering could be a good solution.

# Failed approaches

We did some attempts with the current API, but none of them had a good result.

### chat.update

We tried adding new blocks with [chat.update](https://api.slack.com/methods/chat.update), but the blocks can't be customized by user so the result in a channel with many
participants could be that everybody see a huge list of new blocks containing useless info but one. More than this the users sending messages would need to authorize our
App to send messages on behalf of them, and we couldn't ask everybody (`U0000000001` in the given example) for that authorization...

### chat.postEphemeral

We tried with [chat.postEphemeral](https://api.slack.com/methods/chat.postEphemeral) as it is customized by users, but ephemeral messages have two flows by definition:

* their delivery is not guaranteed
* they do not survive across client restarts

That means we need to add a shortcut to force the app to resend the useful info.

More than this, due to the asynchronous nature of the APIs, it could happen that other messages are posted between the `target message` and the ephemeral message, this
makes it hard to bind the two messages. A failed approach was to attach to the ephemeral message the first part of the `target message`, the result was that the
ephemeral messages were consuming much more vertical space than the original messages. Not to mention what happens when `U0000000001` edits the `target message`...
ephemeral messages can't be updated: posting a new ephemeral message with updated info results in a complete chaos.

### DM between the App and the users

The attempt with the most graceful result was this one, but it had the result with the worst feeling. The useful info is posted in another channel so at the end the result is that the useful info is lost somewhere else rather than posted where it is required.

### Clone each channel for each user

Actually this is not a failed attempt, this is only a never implemented idea. A possible solution could be to make a clone of each channel for each user, `U0000000002`
needs to chat in the dedicated channel clones while the App have to post `U0000000002`'s messages in the original channels and to post (in the `U0000000002` dedicated
channel clones) clones of the messages posted by other users in the original channel, enriching them with the info customized for `U0000000002`. We didn't implemented it
because it would require a considerable effort and it would be a resources consuming solution.
