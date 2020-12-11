============================
Sportstalk 247 Documentation
============================
.. autoclass::
.. autofunction::
.. autoexception::

REST API Reference
------------------
The REST API is documented with Postman.  You can access it here:
`<https://apiref.sportstalk247.com/>`_


Client SDKs
------------------
We provide SDKs for the following languages:

* `Javascript/Node.js (with Typescript Support) <https://sportstalk-sdk-javascript.readthedocs.io/en/latest/>`_
* `Swift (iOS) <https://sportstalk-ios-sdk.readthedocs.io/en/latest/>`_
* `Kotlin (Android) <https://sdk-android-kotlin.readthedocs.io/>`_

The client SDKs are wrappers around the REST API and provide convenience functionality to make development easier.  It is always possible to drop back into the REST API directly if you need special functionality.

The iOS and Android SDKs currently contain only chat functionality.  The Javascript SDK has clients for chat and comments.
In all platforms it's possible to use the REST API directly.

==========================
Getting Started With Chat
==========================

CONCEPTS
--------

* EVENTS are emitted whenever anythign happens in the room.  There are two categories of events:

  * DISPLAYABLE events are events you can see, such as the user said something, entered the room, left the room, an advertisement
  * NOTIFICATION events are events that you can't see, but change the state of the room. For example, if a user says something that will emit a "SPEECH" event. But if a moderator deletes that event, a "REMOVE" event will be emitted with the ID of the speect event.  The SDK user will then remove the event from their app.

* Chatting happens within rooms.  You can configure how each room behaves.

  * PREMODERATED rooms require all displayable messages to go through moderation before they appear in the chat experience
  * POSTMODERATED rooms allow messages to show up immediately, but people chatting can flag events, and if that happens, they will be sent to the moderation queue.

* Details methods in the API return a single object
* List methods in the API return a cursor object

CURSORING
---------
The API uses cursoring which is different from pagenation.  The idea of the cursor is you can call the list function without a cursor to get an initial result set, and then call it again to read more items from the list. This way, as more data is flowing into the system, you can always continue from where you left of to read the latest data.

All of the list method return a cursor object.  The cursor object contains:

* A list field that varies by type, can be a list of comments, events, users, rooms, conversations etc.
* If you have reached the end of the list you are querying:

  * "more" field: This will be set to false.
  * "cursor" field: If the response returns any items, a new cursor string will be returned. If the response returns no items, you will receive back the same cursor you passed in, so you can use it to poll for more data.

* If there are more items to read:

  * "more" field: This will be set to true telling you that you can retrieve more items immediately with another call to the list method.
  * "cursor" field: A new cursor string will be returned which you pass into the next call to the list method to get the next batch of results.

The first time you invoke a list method you do not pass a value for the cursor field. The list method will return an initial list. Call the list method again to get the next set of results.

Example (JavaScript)
--------------------

.. code-block:: javascript

    let more = true;
    let cursor = "";
    while (more) {
        let roomCursor = chatClient.listRooms(cursor, 100);
        roomCursor.rooms.each(function(room) {
            // Do something with the room object
        });
        more = roomCursor.more;
        cursor = roomCursor.cursor;
    }

HOW TO IMPLEMENT A CHAT APP
---------------------------

1. Initialize your SDK.  iOS, Android (Kotlin), and JavaScript SDKs are available.
2. CREATE a room.

   * You can optionally provide a unique, URL friendly custom ID for your room

3. JOIN the chat room.  The user must "JOIN" the room before the user can interact with the room.

   * There are two versions of JOIN: Join by RoomID, and Join by CustomID
   * You can create/update your user with the JOIN command by passing optional parameters
   * JOIN will return information about the room object
   * JOIN will return recent events. JOIN will only return *DISPLAYABLE EVENTS*
   * You can request JOIN also returns threaded replies to recent messages

4. CATCH UP your chat room by passing each event returned by JOIN to your event handler.
5. START LISTENING for updates. This will cause your event handler to be invoked whenever a new event is emitted in your room.
6. EXECUTE commands in the room. For example the command "Hello, World!" will emit a speech event containing "Hello, World!"
7. STOP LISTENING for updates. Your app will stop retrieving updates for the room.
8. EXIT the room. This removes your user from the list of participants.

BEHAVIOR OF JOIN, GET UPDATES, LIST PREVIOUS EVENTS, LIST EVENT HISTORY
------------------------------------------------------------------------

The normal use case for a chat application when you open a chat room is you:
1. Join the room
2. Receive previous events (displayable events that were emitted in the room before you joined) so the user has some context of the conversation
3. Subscribe for notifications of new events
4. Scroll back to view message history using listPreviousEvents

* JOIN: This method will add a user to a room. You can create or update an existing user with this method.  It will also return recent displayable events from before the user joined the room.

  * Join will include an eventsCursor object with recent events in it. The cursor has the same result as if you called getUpdates with no cursor string.

* GET UPDATES:

  * The purpose of getUpdates is to receive events that update the state of the room as users interact with it.
  * When called without a cursor string:

    * This is what join uses internally when returning the eventsCursor.
    * This will return recent displayable events only (non-displayable event types are not returned)
    * The 'more' property will be true if there are previous displayable events, which indicates *scrollback* content is available

  * When a cursor string is provided:

    * getUpdates will get events emitted that are newer than the cursor value even if they are not displayable (like remove, replace, report, react).
    * When you call getUpdates, you are moving forward in time, getting events newer than the cursor in the sort order.
    * You should not try to create your own cursor string, it should only have meaning to the API. It represents a place in a sort order, and how that is calculated could be changed.
    * The SDK manages this for you, invoking getUpdates and tracking the most recent cursor value
    * The 'more' property will be true if there are newer events in the update stream that you can retrieve

* LIST PREVIOUS EVENTS:

  * This method is used to power a scroll back feature. It allows you to look back in time at previous events. When you first call it, it will start from the most recent event, and move backwards in time each time you pass the cursor string.
  * This method only returns displayable events.

* LIST EVENT HISTORY:

  * This method is used to download or export events.
  * The first time you call it without a cursor string it will return the first event in the room and it moves forward in time.
  * It will return all events, regardless of if they are deleted or moderator rejected, or if the event is not displayable.

INTERPRETING EVENT PROPERTIES
-----------------------------

+---------------------------------------------------------------------------------------------------------------------------------+
| PROPERTY           | MEANING                                                                                                    |
+====================|============================================================================================================+
| id                 | The ID for the event assigned by the server                                                                |
| applicationid      | The ID of the application the event is for                                                                 |
| roomid             | The ID of the room in which the event was emitted                                                          |
| body               | The body of the event, used by displayable events like "speech" or quoted reply.                           |
| originalbody       | The original body of the event, as sent by the user. This is empty if the body was not modified. The body  |
|                    | is modified if censoring is enabled and the body contained profanity, or if a moderator altered the body.  |
| deleted            | The event is logically deleted, but it has threaded replies that may be displayed                          |
| added              | When the event was emitted. ISO 8601 time format, in UTC time.                                             |
| modified           | When the event was lasted modified. ISO 8601 time format, in UTC time.                                     |
| ts                 | The high resolution timer (ticks) timestamp of when this event was emitted.                                |
| eventtype          | The type of event this is. See event types below.                                                          |
| userid             | The userid of the user who created this event.                                                             |
| user               | A copy of the user object of the user who created the event.                                               |
| customtype         | Specify a string for your own custom type.                                                                 |
| customid           | Optionally provide a unique key for your event. Must be unique within a room.                              |
| custompayload      | Optionally provide your own JSON or XML object or string.                                                  |
| customtags         | Optionally provide an array of strings to tag your event with.                                             |
| customfield1       | Optionally store any string in this field.                                                                 |
| customfield2       | Optionally store any string in this field.                                                                 |
| replyto            | A copy of the object this event is directed at.                                                            |
| parentid           | If this is a threaded reply this field indicates the parent event.                                         |
| hierarchy          | This is an array, in order, of the parent hierarchy of a threaded reply.                                   |
| depth              | A numeric value indicationg how deep in a tree this item is.                                               |
| editedafterpublish | True if the user modified her message after it was published.                                              |
| editedbymoderator  | True if a moderator changed the contents of the message.                                                   |
| censored           | True if profanity filter censored the content body.                                                        |
| isactive           | False if the event should nto be used or displayed, for example if it is in the moderation queue.          |
| mutedby            | An array of userids of users who have muted the user who sent this event.                                  |
| shadowbanned       | True if the user who posted this event is shadowbanned. Only show that user this event.                    |
| hashtags           | A list of hashtags detected in the message.                                                                |
| likecount          | The number of LIKE reactions on the mesage.                                                                |
| replycount         | The number of replies on the message.                                                                      |
| reactions          | The reactions by users to the event, grouped by reaction type.                                             |
| moderationstate    | Used by moderation queue.                                                                                  |
| reports            | Submissions by users indicating this event is abuse.                                                       |
+--------------------+------------------------------------------------------------------------------------------------------------+

EVENT TYPES
-----------
+--------------+-----------------------------------------------------------------------------------------+-------------+
| EVENT TYPE   | HOW TO REACT TO THIS EVENT TYPE                                                         | DISPLAYABLE |
|--------------|-----------------------------------------------------------------------------------------|-------------|
| unknown      | Ignore this event                                                                       | NO          |
| custom       | Indicates this is a custom event. In this case user is providing a                      | YES         |
|              | custom type value.                                                                      |             |
| announcement | Indicates the message is an announcement                                                | YES         |
| speech       | The user said something in chat                                                         | YES         |
| reply        | The user issued a threaded reply to another event. Display the reply.                   | YES         |
| quote        | The user quoted something someone else said in another event.                           | YES         |
|              | Display the quoted reply.                                                               |             |
| reaction     | The user reacted to an event previously emitted. Update that event in your UI           | NO          |
|              | to display the updated reaction information.                                            |             |
| replace      | The event is replaced with a new version of the event. The updated copy is in           | NO          |
|              | the replyto field. This is only emitted for displayabe event types.                     |             |
| remove       | The referenced event (via parentid field) is to be removed from the chat stream UI      | NO          |
| purge        | Remove all events from the specified user                                               | NO          |
| ad           | Display an advertisement                                                                | YES         |
| enter        | A user entered the chat room                                                            | YES         |
| exit         | A user exited the chat room                                                             | YES         |
| roomopened   | The room has been opened. Display a message that the room is now open.                  | YES         |
| roomclosed   | The room is not accepting any more commands. Display a message that the room is closed. | YES         |
| action       | User performed an action command.                                                       | YES         |
+--------------+-----------------------------------------------------------------------------------------+-------------+


THREADED REPLIES
++++++++++++++++
You can provide the user with the option to post a threaded reply.  This would typically be done in a user interface by having a reply button under a mesage, and when the user replies it starts a tree like structure in the user interface, or a side panel for the a "sidebar" style conversation.  Up to two levels of depth are allowed.  The top level event is considered the "parent" and the replies are considered "children".
* To send a threaded reply, see the SDK documentation.

DELETING EVENTS
+++++++++++++++
* **LOGICAL DELETE** : Since events can have threaded replies, it is possible to delete a parent event and not delete the replies. In this case the event is "logically deleted". When you logically delete an event, its children are not deleted.
* **PERMANENT DELETE** : If you permanently delete an event it is deleted, and so are any threaded replies to that event, and they cannot be recovered.
* You can issue a logical delete command, with the *permanentifnoreplies* flag set to true, which will cause the API to logically delete the event if there are threaded replies, but permanently delete the event if there are no threaded replies.
* If you use the flag *permanentifnoreplies*=true and you delete a reply, if its parent no longer has any replies and is also logically deleted then the parent will also be permanently deleted.
* If an event is permanently deleted, a REMOVE event will be raised in the room notifying you to remove the deleted event from your user interface.
* If an event is logically deleted, it will be modified, and a REPLACE event will be sent to update it.
