# Real-time Messenger Design Challenge

## Context

In 2024, a recruiter reached out from a company that operates a live-streaming
video site, similar to Twitch. I applied for their Principal Engineer role, and
was asked to complete the following design challenge within 24 hours, spending
no more than a couple of hours on it.

To protect the integrity of their hiring processes, the company has asked not to
be identified.

After submitting this document, I was asked to complete a virtual interview loop
consisting of four one-hour interviews, each with two members of the team. I was
offered the position and declined.

This repository is made public under the MIT license for educational purposes.

Final Output: [PDF](design_proposal.pdf) [Markdown](design_proposal.md)

---

# Design Proposal: Messenger Feature

This design outlines an approach for a chat application supporting real time and
asynchronous messaging. The high level components are:

- API Gateway & Authorization Service - authorizes the user's credentials. For
  an unauthenticated user, or a revoked / unauthorized credentials token, it is
  assumed there is an existing login service we can redirect the user to.

- Messaging Service - The primary service for opening and persisting a chat
  connection. This is a load-balanced backend service (e.g. NodeJS), deployed
  via a Docker container to an EC2 cluster (or similar).

- List Service - Existing service to provide group messaging lists.

- Billing Service - Existing service to process payments.

- Redis Instance - Used as (1) a cache for tracking which users have active
  connections, and which specific server instances they are connected to. Also
  used as (2) a message queue for processing incoming messages.

- MessageHandler - An asynchronous task handler to process messages and tips,
  and to deliver them to the intended recipients.

- MessagingDB - A SQL database to store the messages and related data. This
  design requires two primary tables:
  1. A `messages` table to store the historical record of messages between
     users. This could include metadata such as read status, read_date, etc.
  2. A `blocked_users` table to track recipients that have blocked specific
     senders.

![Component Diagram](block_diagram.png)

## Real Time Chat

To achieve a real time chat, we will utilize WebSockets, Redis, and an
asynchronous task processor.

### Initiating a Connection

When a user connects from the application (e.g. React application), the Gateway
authorizes the connection via the provided credentials, forwards the request to
an available MessageService instance, and opens the WebSocket connection.

Once the WebSocket connection is open, the MessageService stores an identifier
in a shared Redis cache, so that new incoming messages can locate the server
instance with the active connection.

Recent messages are loaded from the DB, and sent to the user over the newly
established connection.

When the connection is terminated, the server instance removes the stale entry
from the Redis cache.

![Sequence Diagram: Opening a Connection](connection.png)

### Sending Messages

A message is sent via the open WebSocket connection (or via an HTTP endpoint) to
the MessageService. The MessageService does basic filtering steps, such as (a)
ensuring the recipient exists, and (b) ensuring the user is sending to a valid
list belonging to them, if the user is a creator. It is then sent to a queue to
be picked up by the async task processor, MessageHandler.

The MessageHandler will verify the recipient has not blocked the sender, store
the message in the database, and notify the user by routing the message through
MessageService via all active websocket connections.

![Sequence Diagram: Sending and Receiving](sending.png)

## Offline messaging

We achieve offline messaging by storing the messages in a relational database
(e.g. MariaDB), indexed on `recipient_id`, `sender_id`, and `sent_date`. This
step is taken regardless of whether a message is sent, so that messages are not
lost when a user closes the app. When a user logs in, they will receive all
unread messages after the connection is established.

Messages could also be retrieved without engaging in live chat, via an HTTP
endpoint that queries the MessageDB (not represented in diagram).

ref: See the Sending Messages sequence diagram above.

Out of Scope: Email, SMS, or other notifications could be sent when an offline
user recieves a message by sending a notification after storing the new message
in the database.

## Mass messaging

Mass messaging is achieved through the same flow as individual messaging, with
two added steps: When the message is sent, before being queued for processing,
authorization checks are made against the sender and recipient list. The
MessageHandler expands the list of recipients, and sends the message to each
recipient via the MessageService instance with the active websocket connection.

ref: See the Sending Messages sequence diagram above.

## Tipping

When a member tips a creator, it can be sent through the active websocket
connection or via an HTTP endpoint. The request is made to the billing service -
on success, the creator is notified with a message via the existing message
flow. On failure, the member will receive a message with a link redirecting them
to the Billing preferences to update their payment information.

It is assumed that tipping will be sent through the messaging flow, and that
tips may include a message. An optional "tip amount" attribute is added to the
request, and processed through the Billing service if it is nonzero.

ref: See the Sending Messages sequence diagram, this flow is outlined in the
async loop section when processing messages.

## Blocking

A user can block another user through an API call, which updates a
`blocked_users` table in the MessageDB. Messages sent to a blocked user will not
be processed. It is assumed that the user will not be notified that their
messages are blocked, they will just be sent into the void.

To notify the user that their message has been blocked, we could instead do the
block lookup when the message is received, before queueing it for processing.

Note: Blocked messages may be logged for investigative purposes and to prevent
abuse. An alternate approach could also store the messages with a flag
indicating it was blocked. Those messages would not be sent to the recipient,
and would be filtered out when retrieving messages from MessageService (not
represented in diagram)

## Additional Considerations

- Integration tests and unit tests should be written as a part of the
  implementation.
- All services should include a Dockerfile with a release build.
- Infrastructure as Code (IaC) scripts should be written todefine the
  infrastructure and deployments.
- CICD workflows should be included to Build, Test, Release, and Deploy.
- Basic security controls are handled with the Authorization service and API
  Gateway, but more rigor and testing may be required.
- As traffic grows, to reduce strain on the primary DB an additional Redis cache
  could be added to cache frequently accessed data, such as block lists for
  high-traffic creators, messages for frequent users, or messages sent to very
  large groups.

[Copyright Nik Gilmore 2024 All Rights Reserved]

---

## Feedback

I received the following feedback from the recruiter. I haven't responded to the
feedback, but agree with many of the points made. Please understand that this
project was done with a restricted time window, and was done in my personal free
time. It is a first draft, and I would have had the opportunity to address these
concerns in a real-world situation involving team discussions and an iterative
process.

    Please note that you received amazing feedback,
    but I wanted to share a few things that were mentioned just in case it is
    helpful! Also, I wanted to note that the two individuals who reviewed your
    design challenge will not be a part of your interview loop

    - Minor - Database design would have been nice; it was vaguely described in the
      text.

    - Minor - Timeout handling, failure management, scaling and monitoring could
      have been mentioned. These could be discussed on the interview.

    - Minor - The solution can be good enough for simple use cases, but it is not
      scalable enough. With this design approach, user connections for the same
      group are spread across nodes, so every message has to be delivered to every
      Messaging Service node before being sent to the individual recipients. In the
      case of a high user/message count, those nodes could be easily overloaded.

    - Minor - A better approach would be to use a consistent hash ring when
      selecting where the user connections are being routed. This way, the
      connections that have to talk to each other could be routed to the same node,
      and the message would never have to leave that node.

    Initiating a Connection:

    - Request forwarding: Will the WebSocket connection be proxied through the API
      Gateway, or will the user directly connect to the Messaging Service? Both
      approaches have their pros and cons.

    Sending messages:

    - If recipient_type == "group", we should check if the user is a creator before
      retrieving the lists

    Offline messaging:

    - An index is missing on recipient_type • Why do we save the message for each
      user instead of just saving it once using the group id? Messages from blocked
      users could be filtered out before delivery.

    Tipping:

    - The first part of the sequence diagram makes it clear that each group belongs
      to exactly one creator. Why are we attempting to process the payment for each
      user in the group instead of simply attempting to process it once using the
      creator ID?

## LICENSE

Copyright 2024 Nik Gilmore

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the “Software”), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
