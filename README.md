Messaging Architecture
======================
High level design for real time sending of actions

There are 3 main components to the system:
 - Client Managers
 - Centralised Controller
 - Amazon MSK

There are 3 basic use cases covered:
 - Client connections
 - Sending of actions
 - Action results


Architecture Diagram
====================

![alt text](https://github.com/emaley/messaging-part2/raw/master/images/realtime-messaging-architecture.jpg "Architecture Diagram")


Components
====================


Client Managers
-------------------
Client Managers roles are:
 - Receive connections from clients and maintain a key value store of connected clients
 - Authorise users before to sending actions to clients
 - Action Results passed onto MSK




Centralised Controller
-------------------
 - Maintains a key value store of connected clients and associated Client Managers
 - Receives action requests from users and sends to MSK for actioning to clients that are connected
 - Manages sent actions and received results



Amazon MSK
-------------------
 - Provides topics for clients connecting (subscriptions), sending actions, results from actions and dead letter actions
 - MSK provides the consistency and reliability for the sending of events


Use Cases
====================


Client connections
-------------------
 - Client requests JWT from `Authorisation Server`
 - Use SSE to create a connection to the _Client Manager_
 - The ELB load balancer uses LOR for load balancing requests to available _Client Managers_
 - _Client Manager_ validates JWT token
 - _Client Manager_ keeps track of connected clients via key value store (Redis)
 - _Client Manager_ publishes to MSK `Subscription` topic for _Centralised Controller_
 - Messages published to the `Subscription` topic contain the `ActionCM(X)` topic name that is unique to the specific _Client Manager_ eg `ActionCM1`
 - _Centralised Controller_ consumes `Subscription` topic to know which clients are connected to which _Client Manager_
 - _Centralised Controller_ keeps track of connected clients and which _Client Manager_ they are connected to


Sending of actions
-------------------
 - Users request actions to be sent to a set of devices
 - _Centralised Controller_ checks that client is connected and then publishes to the _Client Manager_ specific MSK `ActionCM(X)` topic
 - _Client Manager_ consumes it's own `ActionCM(X)` topic and checks that it has the client in Redis
 - Checks that the user requesting the action is authorised via the `User Authorisation` service
 - Sends action to the client


Action results
-------------------
 - Client sends result of action back to the server either through connection if bi directional communication is enabled, or through a REST API
 - _Client Manager_ forwards action result onto MSK `Action Result` topic
 - _Centralised Controller_ marks action as completed