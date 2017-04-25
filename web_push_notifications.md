# Web Push Notifications


Web Push Notifications requires a service worker to run in the browser through which push notification can be sent. It sends the notification through Firebase Cloud Messaging enabling applications to interact with the users even while not using the application.

 # Registering application on Firebase
 To send Web Push Notification to the client, you need to register your application on Firebase Console. For registering a new project follow the following steps:
- Register a new project on Firebase.
- Go to Settings > Project Settings.
- Go to Cloud Messaging Tabs.
- Save the server key and sender id. Sender id will be used while registering service worker on client’s machine and server key will be used to send push notification. Sender Id will be stored in the manifest.json file and server key will be used to send chrome push notification from the server.

# Setting up service worker
- Create a manifest.json file which will contain the information about the application.
 ```
{
"name": <Application Name>,
"gcm_sender_id": <Sender id from the Firebase registered project>
}
```
 - Create a service-worker.js file which will contain the event listeners and configuration for notification. 
Push Event listener will push the notification to the user when an event is received by Service worker. 		

```
// Register event listener for the 'push' event.
self.addEventListener('push', function(event) {

 payload = event.data.text();

 // Keep the service worker alive until the notification is created.
 event.waitUntil(
   // Show a notification with title 'ServiceWorker Cookbook' and use the payload
   // as the body.
   self.registration.showNotification(payload.title, payload.options)
 );
});
```
- Notification click event listener specifies the action that needs to be performed when a notification is clicked. 

```
// Event Listener for notification click
self.addEventListener('notificationclick', function(event) {

 event.notification.close();

 event.waitUntil(
   clients.openWindow(<URL>)
 );
});

```


- Subscribing/ Unsubscribing User

Create a javascript file which contains the code for subscribing to push notification, adding service worker, asking user for notification permission and storing the endpoint in the database. 

Define the following variables
```
var serviceWorkerSrc = "/static/serviceworker.js";
var callhome = function(status) {
  console.log(status);
}
var storage = window.localStorage;
var registration;
```


Define onPageLoad() which will check and register the service worker. On successful registration of service worker it will call initialiseState() which is defined in the next step.

```
 function onPageLoad() {
   // Do everything if the Browser Supports Service Worker
   if ('serviceWorker' in navigator) {
     navigator.serviceWorker.register(serviceWorkerSrc)
       .then(
         function(reg) {
           registration = reg;
           initialiseState(reg);
         }
       );
   }
   // If service worker not supported, show warning to the message box
   else {
     callhome("service-worker-not-supported");
   }
 }
```

Define initialiseState() which will check whether notifications are supported by the browser or not. If notifications are supported it will call the subscribe() which is defined in the next step.

```
 // Once the service worker is registered set the initial state
 function initialiseState(reg) {
   // Are Notifications supported in the service worker?
   if (!(reg.showNotification)) {
     callhome("showing-notifications-not-supported-in-browser");
     return;
   }

   // Check the current Notification permission.
   // If its denied, it's a permanent block until the
   // user changes the permission
   if (Notification.permission === 'denied') {
     callhome("notifications-disabled-by-user");
     return;
   }

   // Check if push messaging is supported
   if (!('PushManager' in window)) {
     // Show a message and activate the button
     console.log('push-notifications-not-supported-in-browser');
     return;
   }
   subscribe();
 }
```
Define subscribe() which try to subscribe the user to push notification. If subscription does not exist, it will ask user for notification permission. Once notification permission is granted, it will receive the subscription details which will be sent to postSubscribeObj() for processing. If user has already subscribed the same subscription detials will be reuturned.

```
 function subscribe() {
   registration.pushManager.getSubscription().then(
       function(existing_subscription) {
         // Check if Subscription is available
         if (existing_subscription) {
           endpoint = existing_subscription.toJSON()['endpoint']
           if (storage.getItem(endpoint) === 'failed') {
             postSubscribeObj('subscribe', existing_subscription);
           }
           return existing_subscription;
         }
         // If not, register one using the
         // registration object we got when
         // we registered the service worker
         registration.pushManager.subscribe({
           userVisibleOnly: true
         }).then(
           function(new_subscription) {
             postSubscribeObj('subscribe', new_subscription);
           }
         );
       }
     )
 }
```

Define unsubscribe() which will unsubscribe user from the push notification by deleting the endpoint from the database. It will get the subscription details from the pushManager and send it to postSubscribeObj() to update the subscription status on the server.
```
 function unsubscribe() {
   navigator.serviceWorker.ready.then(function(existing_reg) {
     // Get the Subscription to unregister
     registration.pushManager.getSubscription()
       .then(
         function(subscription) {
           // Check we have a subscription to unsubscribe
           if (!subscription) {
             return;
           }
           postSubscribeObj('unsubscribe', subscription);
         }
       )
   });
 }
```
Define postSubscribeObj() which updates the endpoint on the server by making an API call. 
```
 function postSubscribeObj(statusType, subscription) {
   // Send the information to the server with fetch API.
   // the type of the request, the name of the user subscribing,
   // and the push subscription endpoint + key the server needs
   // to send push messages

   var subscription = subscription.toJSON();
   // API call to store the endpoint in the database
  
 }
```
Call onPageLoad() to initialize the worker status check and updating subscription.
```
 onPageLoad();
```


- Add above javascript file and manifest.json on the home page where you want to ask notification permission from the user.
- If the user allows, the push notification will be subscribed and the endpoint can be stored. On successful registration, add an API call to save the notification endpoint which will be used to send web push notification to the user.
# Sending Web Push Notification
On successful subscription, endpoint will be stored which can be used to send notification to the user. Endpoint if in the following format:
For python, Pywebpush package can be used to send web push notification.
	
    The subscription JSON will be received on successful subscription.
    GCM_KEY - The server key from the Firebase console
    ttl - Time to Live (Integer)

```
from pywebpush import WebPusher

subscription = {
   "endpoint": endpoint,
   "keys": {
       "auth": auth,
       "p256dh": p256dh
   }
}
payload = {
   "title": title,
   "options": options or {}
}
WebPusher(subscription).\
   send(json.dumps(payload), {}, ttl, GCM_KEY)
```



### References
https://developers.google.com/web/fundamentals/getting-started/codelabs/push-notifications/

Push Libraries- https://github.com/web-push-libs/


