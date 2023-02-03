
# TL;DR

When a connection is established between an issuer and holder via a mediator there is a period when the connection is in state `"rfc23_state": "response-sent"` and  `"rfc23_state": "completed",` when a message, such an an offer, can be sent by the issuer to the holder. If this happens the offer is queued in the mediator and no re-delivery attempt is made.

# Description

There appears to be a condition where, after running for some time, an ACA-py mediator can enter a state where:

 When a connection is established between an issuer and holder via a mediator there is a period when the connection is in state `"rfc23_state": "response-sent"` and  `"rfc23_state": "completed",` when a message, such an an offer, can be sent by the issuer to the holder. If this happens the offer is queued in the mediator and no re-delivery attempt is made.

 This appears to be a race condition where the mediator performance slows just enough (on cloud infrastructure) to allow the above mentiond scenario to serface. It can be reproduced using a local mediator by, as the description notes, sending an offer quickly befor a connection state reaches the "completed" state.

 # Steps to Reproduce

### Setup

 1. Start an [ACA-py mediator](https://github.com/hyperledger/aries-mediator-service) locally or otherwise;
 2. Start the [ACA-py Faber](https://github.com/hyperledger/aries-cloudagent-python);
 3. Start either the sample script or a fresh install of the BC Wallet.

### Trigger the Issue

1. Using a **fresh install** of the BC Wallet scan Faber's invitation QR code;
2. As soon as the Faber UI displays the first connection message, show below,
press #1 to offer a credential.

```JSON
{
    "their_label": "BC Wallet",
    "connection_protocol": "connections/1.0",
    "updated_at": "2023-02-02T20:17:34.320044Z",
    "my_did": "KPZpDbMPDXZYVZEKRttanB",
    "connection_id": "b58b186b-d4ac-439c-9eb4-93ed38ba6eef",
    "rfc23_state": "response-sent",
    "invitation_key": "EAUsTQpExyNa7LpwQtDgdxbbWASA4ubMBhAGeGbV1uzX",
    "routing_state": "none",
    "invitation_mode": "once",
    "accept": "auto",
    "their_role": "invitee",
    "created_at": "2023-02-02T20:13:32.734545Z",
    "their_did": "4AM74FNHKm3RCHq327rMnt",
    "state": "response"
}
```
3. The offer will be sent over the conneciton before it is fully setup resulting
in the message being queued on on the mediator.
4. The following message will then be displayed in Faber indicating the connection
is fully setup:

```JSON
{
    "their_label": "BC Wallet",
    "connection_protocol": "connections/1.0",
    "updated_at": "2023-02-02T20:17:37.660641Z",
    "my_did": "KPZpDbMPDXZYVZEKRttanB",
    "connection_id": "b58b186b-d4ac-439c-9eb4-93ed38ba6eef",
    "rfc23_state": "completed",
    "invitation_key": "EAUsTQpExyNa7LpwQtDgdxbbWASA4ubMBhAGeGbV1uzX",
    "routing_state": "none",
    "invitation_mode": "once",
    "accept": "auto",
    "their_role": "invitee",
    "created_at": "2023-02-02T20:13:32.734545Z",
    "their_did": "4AM74FNHKm3RCHq327rMnt",
    "state": "response"
}
```

4. At this point nothing will show up in the wallet. Returning to the home screen 
will display no notifications (offers). 

### Same Senario, No Issue

1. Using a **fresh install** of the BC Wallet scan Faber's invitation QR code;
2. You will see a message similar to the following in the Faber UI, **do nothing**:

```JSON
{
    "their_label": "BC Wallet",
    "connection_protocol": "connections/1.0",
    "updated_at": "2023-02-02T20:17:34.320044Z",
    "my_did": "KPZpDbMPDXZYVZEKRttanB",
    "connection_id": "b58b186b-d4ac-439c-9eb4-93ed38ba6eef",
    "rfc23_state": "response-sent",
    "invitation_key": "EAUsTQpExyNa7LpwQtDgdxbbWASA4ubMBhAGeGbV1uzX",
    "routing_state": "none",
    "invitation_mode": "once",
    "accept": "auto",
    "their_role": "invitee",
    "created_at": "2023-02-02T20:13:32.734545Z",
    "their_did": "4AM74FNHKm3RCHq327rMnt",
    "state": "response"
}
```
3. Wait for the following messge to be displayed in the Faber UI:

```JSON
{
    "their_label": "BC Wallet",
    "connection_protocol": "connections/1.0",
    "updated_at": "2023-02-02T20:17:37.660641Z",
    "my_did": "KPZpDbMPDXZYVZEKRttanB",
    "connection_id": "b58b186b-d4ac-439c-9eb4-93ed38ba6eef",
    "rfc23_state": "completed",
    "invitation_key": "EAUsTQpExyNa7LpwQtDgdxbbWASA4ubMBhAGeGbV1uzX",
    "routing_state": "none",
    "invitation_mode": "once",
    "accept": "auto",
    "their_role": "invitee",
    "created_at": "2023-02-02T20:13:32.734545Z",
    "their_did": "4AM74FNHKm3RCHq327rMnt",
    "state": "response"
}
```

4. Once this message is displayed, press #1 to offer a credential.
5. At this point the offer will show up in the wallet. Returning to the home screen 
will display the notification (offer). 

### Expected Behaviour

[RFC 0160](https://github.com/hyperledger/aries-rfcs/blob/main/features/0160-connection-protocol/README.md) does not require a acknowlwdgement that a connection is completed before message can be sent over it. This is address in V2. 

An ACA-py mediator should atempt delivery of any queued messages when the related connection becomes "completed" to remediate this issue.

### Q & A

1.
Q. How do you know the message is queued in the mediator?
A. In AFJ the fn `initiateMessagePickup` can be called to trigger the delivery of messages. The outstanding offer will be delivered. 

2.
Q. Is this infrastructure (OpenShift, Cloud, Kubernets) related?
A. This problem exists on two mediators running similar version of ACA-py hosted on different infrastructure by two different companies. It can also be reproduced locally using Docker.

3.
Q. Is this specific to a version of ACA-py. 
A. It can be reproduced locally in Dokcer using ACA-py 0.7.3 and 1.0.0-rc1. The cloud hosted agents both run ACA-py 0.7.x versions.

4.
Q. Why do you think its a race condition?
A. In one test we used an ACA-py 0.7.3 mediator on a cloud platform which had been running for 1 day under light load. The problem was evident. On the same cloud platform an ACA-py mediator running 0.7.4-rc2 which had been running for <10 min. The problem did not present. This leands us to believe that as a mediator is used performance degrades enough for the problem to present.

5.
Q. Could this be the issuer rather than the mediator?
A. Unlikley. The situation can be reproduced using the BC Showcase demo. By using the older mediator mentiond in #4 above the automated showcase demo fails. By using the fresh mediator from #4 above the demo succeeds.

6.
Q. What conneciotn protocol is being used?
A. V1. 

7. 
Q. Is this a bug?
A. Maybe. [RFC 0160](https://github.com/hyperledger/aries-rfcs/blob/main/features/0160-connection-protocol/README.md) does **not** require acknowledgement when a conneciton enters the "completed" state. Its is by convention that a controller should confirm the state before sending a message (offer) over the conneciton. This is a recongized shortcoming that shoudl be addressed in V2.

However, it may be a bug in that an ACA-py mediator does not atempt re-delivery of queued messages if that message comes in before the conneciton is "completed". 
