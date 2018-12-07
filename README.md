# Part 4: MicroProfile Meeting Application - Using WebSockets and CDI events

## Overview
In this lab we will cover adding WebSockets and using CDI Events to integrate a WebSocket and the CDI beans so that the server can notify clients about changes

If you have eagle eyes, you might have noticed with the application that the redirect only works when a meeting is started, not when it is already running. In order to make the join action work, the browser needs to find out when the meeting has changed. This can be done by polling the server but that can be expensive. Instead, a WebSocket can be used to allow the server to notify the client. This reduces the number of requests to the server and provides prompt updates to the client.

Adapted from the blog post: [Writing a simple MicroProfile application: Using WebSockets and CDI events](https://developer.ibm.com/wasdev/docs/simple-microprofile-application-using-websockets-cdi-events/)

## Prerequisites
 * Completed [Part 3: MicroProfile Meeting Application - Using Java EE Concurrency](https://github.com/IBM/microprofile-meeting-concurrency)
 * [Eclipse IDE for Web Developers](http://www.eclipse.org/downloads/): Run the installer and select Eclipse IDE for Java EE developers. **Note:** these steps were tested on the 2018-09 version of Eclipse running on Linux and Liberty Developer Tools 18.0.0.3.  **Note:** If you encounter an error message like  `Could not initialize class org.codehaus.plexus.archiver.jar.JarArchiver` please see the Troubleshooting section.
 * IBM Liberty Developer Tools (WDT)
   1. Start Eclipse
   2. Launch the Eclipse Marketplace: **Help** -> **Eclipse Marketplace**
   3. Search for **IBM Liberty Developer Tools**, and click **Install** with the defaults configuration selected
 * [Git](https://git-scm.com/downloads)
 * Install the [IBM Cloud CLI](https://console.bluemix.net/docs/cli/index.html#overview) 
## Steps
### Step 1. Check out the source code

#### From the command line
Run the following commands:
  
```
$    git clone https://github.com/IBM/microprofile-meeting-websockets.git
```

#### In Eclipse, import the project as an existing project.
1. In Eclipse, switch to the Git perspective.
2. Click **Clone a Git repository** from the Git Repositories view.
3. Enter URI `https://github.com/IBM/microprofile-meeting-websockets.git`
4. Click **Next**, then click **Next** again accepting the defaults.
5. From the **Initial branch** drop-down list, click **master**.
6. Select **Import all existing Eclipse projects after clone finishes**, then click **Finish**.
7. Switch to the Java EE perspective.
8. The meetings project is automatically created in the Project Explorer view.

### Step 2. Installing MongoDB
If you completed the previous labs and installed MongoDB, make sure MongoDB is running. If you are starting fresh, make sure you install MongoDB. Depending on what platform you are on the installation instructions may be different. For this exercise you should get the community version of MongoDB from the [mongoDB download-center](https://www.mongodb.com/download-center#community).

1. Once installed you can run the MongoDB database daemon using:
```bash
mongod -dbpath <path to database>
```

The database needs to be running for the application to work. If it is not running there will be a lot of noise in the server logs.

### Step 3. Updating the application to compile against the WebSocket API
To start writing code, the Maven `pom.xml` needs to be updated to indicate the dependency on the WebSocket API for Java EE:

1. Open the `pom.xml` in Eclipse.
2. In the editor, select the **Dependencies** tab.
3. On the **Dependencies** tab there are two sections, one for Dependencies and the other for Dependency Management. Just to the right of the Dependencies box there is an **Add** button. Click the **Add** button.
4. Enter a **groupdId** of `javax.websocket`.
5. Enter a **artifactId** of `javax.websocket-api`.
6. Enter a **version** of `1.1`.
7. From the `scope` drop-down list, select **provided**. This will allow the application to compile but will prevent the Maven WAR packager putting the API in the WAR file. Later, the build will be configured to make it available to the server.
8. Click **OK**.
9. Save the `pom.xml`.

### Step 4. Create a CDI qualifier
A CDI qualifier is simply an annotation annotated with `@Qualifier`. This can then be used with other CDI annotations to influence behaviour. In the case of CDI events, it links the event producer to the event consumer.

1. Right-click the **meetings project, then click New > Annotation…**.
2. Enter a name of `MeetingEvent`.
3. Click **Finish**.
4. The annotations should be added to the type name. There are three key annotations. The first is `Qualifier`, which indicates that the annotation is a CDI qualifier:
```java
@Qualifier
public @interface MeetingEvent {
}
```
5. This introduces a new type `Qualifier` in the package `javax.inject`:
```java
import javax.inject.Qualifier;
```
6. The second annotation, `Retention`, indicates that the annotation should be available at runtime. This allows the CDI runtime to process them:
```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
```
7. This introduces two new types: `Retention` and `RetentionPolicy`. These are in the package `java.lang.annotation`:
```java
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
```
8. The last annotation, `Target`, indicates where the annotation can be applied. For the CDI qualifier it needs to be applied to a field and a parameter:
```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.PARAMETER})
```
9. This introduces two new types: `Target` and `ElementType`. These are in the package `java.lang.annotation`:
```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Target;
```
10. Save the file. The annotations should look like this:
```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.PARAMETER})
public @interface MeetingEvent {
 
}
```

### Step 5. Creating a CDI event object
With CDI events you can pass any object you want between the producer and consumer but, for this application, we will create an object. There are two things to be passed: one is the event to identify the meeting, the other is the URL of the meeting.

To create the event object:

1. Right-click the **meetings project, then click New > Class…**.
2. In the **Package** field, type `net.wasdev.samples.microProfile.meetings`.
3. Enter a name of `MeetingStartEvent`.
4. Click **Finish**.
5. Add a `String` field to store the ID.
```java
private String id;
```
6. Add a `String` field to store the URL.
```java
private String url;
```
7. Next add an constructor that takes the values of the `id` and `url` and stores them in the fields:
```java
public MeetingStartEvent(String id, String url) {
    this.id = id;
    this.url = url;
}
```
8. Finally, create the simple getters to return the fields:
```java
public String getId() {
    return id;
}
 
public String getUrl() {
    return url;
}
```
9. Save the file.

### Step 6. Sending an event when a meeting starts
The next part is to get the `MeetingManager` to emit an event when a meeting is started:

1. Open the **MeetingManager** class.
2. Add a new field to inject the CDI `Event` class. The `Event` class is parameterized with the event object to be set. This field should also be annotated using the CDI qualifier, `MeetingEvent`, that we created earlier:
```java
@Resource
private ManagedScheduledExecutorService executor;
@Inject
@MeetingEvent
private Event<MeetingStartEvent> events;
```
3. This introduces a new type `Event` in the package `javax.enterprise.event`. It also introduces `Inject` from the package `javax.inject`:
```java
import javax.enterprise.event.Event;
import javax.inject.Inject;
```
4. Find the `startMeeting` method. At the end of the method construct a new instance of the `MeetingStartEvent` passing in the meeting ID and URL:
```java
MeetingStartEvent eventObject = new MeetingStartEvent(id, url);
```
5. Then call the `Event` fire object passing in the event object:
```java
events.fire(eventObject);
```
6. Save the file.

At this stage the application could be run, the event would be emitted, but nothing would happen since there is nothing to receive the event.

### Step 7. Creating the WebSocket
The WebSocket will handle the connection between the browser and server, and receive the meeting start event. The browser will send the meeting ID and the WebSocket will notify it when that meeting gets started.

To create the WebSocket:

1. Right-click the **meetings project, then click New > Class…**.
2. In the **Package** field, type `net.wasdev.samples.microProfile.meetings`.
3. Enter a name of `MeetingNotifier`.
4. Click **Finish**.
5. According to the spec, WebSocket components are not CDI beans. To ensure that CDI can see the bean it needs to be annotated. In this case we add the `Dependent` annotation to the type:
```java
@Dependent
public class MeetingNotifier {
}
```
6. This introduces a new class `Dependent` which is in package `javax.enterprise.context`:
```java
import javax.enterprise.context.Dependent;
```
7. To make the class into a WebSocket it needs to be annotated with the `ServerEndpoint` annotation. The annotation takes a URL path that will be used to invoke it. The URL path must start with a forward slash:
```java
@Dependent
@ServerEndpoint("/notifier")
```
8. This introduces a new class `ServerEndpoint` which is in package `javax.websocket.server`. When importing take care to import the right one since there are multiple `ServerEndpoint` classes:
```java
import javax.websocket.server.ServerEndpoint;
```
9. Save the file. The type definition should now look like this:
```java
@Dependent
@ServerEndpoint("/notifier")
public class MeetingNotifier {
```
10. The WebSocket will need to interact with the `MeetingManager` so it needs to be injected into a field:
```java
public class MeetingNotifier {
@Inject
private MeetingManager manager;
```
11. This introduces a new type `Inject` from the `javax.inject` package:
```java
import javax.inject.Inject;
```
12. The WebSocket container manages an instance of the class for each WebSocket connection. When the CDI event system distributes events, however, it creates a new instance so the WebSocket `Session` objects need to be stored for later. A `Map` is used to store the `Session` objects associated with a meeting. Because there will be multiple `Session` objects, a `Collection` of `Session` objects is appropriate. Of course, because this will need to cope with multiple threads, we use concurrent versions of the sessions (added on the next line of the `MeetingNotifier` class):
```java
private static ConcurrentMap<String, Queue<Session>> listeners = new ConcurrentHashMap<>();
```
13. This introduces four new classes. The `ConcurrentMap`, and `ConcurrentHashMap` classes are in the `java.util.concurrent` package. The `Queue` class is in the `java.util` package and `Session` is in the `javax.websocket` package:
```java
import java.util.Queue;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import javax.websocket.Session;
```
14. There are multiple methods on a `ServerEndpoint` but for this the key one is the `onMessage` method:
 * An `onMessage` method is indicated using the `OnMessage` annotation. There are multiple method signatures that can be used but in this case the method will take a String that will contain the ID of the meeting and the WebSocket session:
```java
@OnMessage
public void onMessage(String id, Session s) throws IOException {
    // code will go in here
}
```
 * This introduces a new type the `OnMessage` annotation in the `javax.websocket` package, as well as `IOException` from `java.io`:
```java
import javax.websocket.OnMessage;
import java.io.IOException;
```
 * The first thing to do is to check that the ID really is for a meeting. If there is no meeting with the ID’s name then the method should exit:
```java
JsonObject m = manager.get(id);
if (m == null) {
    s.close();
    return;
}
```
 * This introduces a new class `JsonObject` which is in the package `javax.json`:
```java
import javax.json.JsonObject;
```
 * The next thing to do is to get the meeting URL for the meeting:
```java
JsonString url = m.getJsonString("meetingURL");
```
 * This introduces a new class `JsonString` which is in the package `javax.json`:
```java
import javax.json.JsonString;
```
 * If the meeting URL is there, the information should be sent to the WebSocket client directly then the method should exit. To send information to the client the session is used: get a remote object, then send some text. The `JsonString` `toString` method wraps the URL in quotes so the `getString` method must be used:
```java
if (url != null) {
    s.getBasicRemote().sendText(url.getString());
    s.close();
    return;
}
```
 * Now the session needs to be stored away so that when the meeting is started the client is notified. This is stored in the map, so the first thing to do is to get the collection of sessions:
```java
Queue<Session> sessions = listeners.get(id);
if (sessions == null) {
    // code will go here
}
```
 * Inside the null check we need to create a new collection. This should be a concurrent collection, so use an `ArrayBlockingQueue`:
```java
sessions = new ArrayBlockingQueue<>(1000);
```
 * This introduces a new class, the `ArrayBlockingQueue` in the package `java.util.concurrent`.
```java
import java.util.concurrent.ArrayBlockingQueue;
```
 * Now it needs to be put in the map. Of course there could be two clients coming through the method so, rather than doing a `put` which will overwrite, use the `putIfAbsent` method:
```java
Queue<Session> actual = listeners.putIfAbsent(id, sessions);
```
 * If the `put` succeeded, the `actual` will be null. If another thread won and put their copy of `sessions` in the map it’ll have the collection that should be used, so a swap is needed:
```java
if (actual != null) {
    sessions = actual;
}
```
 * The last thing to do in the method (and outside the if block with the null check for `sessions`) is to add the `Session` to the `Collection` of `Session` objects:
```java
sessions.add(s);
```
 * The code added as a result of steps h-m should look like this:
```java
Queue<Session> sessions = listeners.get(id);
if (sessions == null) {
    sessions = new ArrayBlockingQueue<>(1000);
    Queue<Session> actual = listeners.putIfAbsent(id, sessions);
    if (actual != null) {
        sessions = actual;
    }
}
sessions.add(s);
```
15. Now the sessions are stored, the event method needs to be defined:
 * The name of the method isn’t important but it has to take the event. The parameter that takes the event needs to be annotated with the `Observes` annotation (which indicates that this is an event notification method) and the `MeetingEvent` annotation so it knows which kind of event to call with it:
```java
public void startMeeting(@Observes @MeetingEvent MeetingStartEvent event) {
    // add the notification code here
}
```
 * This introduces the new type `Observes` in the package `javax.enterprise.event`:
```java
import javax.enterprise.event.Observes;
```
 * If this method is called then the meeting has started. The sessions no longer need to be cached away because the meeting has started and so they can be removed from the map:
```java
Queue<Session> sessions = listeners.remove(event.getId());
```
 * Of course it is possible there are no sessions stored, at which point it’ll be null so the next part should only happen if the `sessions` are non-null:
```java
if (sessions != null) {
    // add the next bit of code  here
}
```
 * The logic should be done for each session, so a simple enhanced for loop will do:
```java
for (Session s : sessions) {
    // add the next bit of code here
}
```
 * The session needs to be open to send data to the client, so check that first:
```java
if (s.isOpen()) {
    // add the next bit of code here
}
```
 * Finally the URL should be sent to the client. This could cause an `IOException` which can’t be thrown by this method, so needs to be caught:
```java
try {
    s.getBasicRemote().sendText(event.getUrl());
    s.close();
} catch (IOException e) {
    e.printStackTrace();
}
```
16. Save the file.

You’ve now coded the application. Test the app by opening two browser windows, one to join the meeting and the other to start the meeting. Watch as both browser windows redirect at once.

### Step 8. Configuring Liberty to run WebSockets
1. Open the `server.xml` from **src > main > liberty > config > server.xml**.
2. Find the `<feature manager>` element. It should look like this:
```xml
<featureManager>
    <feature>mongodb-2.0</feature>
    <feature>concurrent-1.0</feature>
</featureManager>
```
3. Before the closing `</featureManager>` element add a `feature` element with the feature `websocket-1.1` as the body.
```xml
<feature>websocket-1.1</feature>
```
4. Save the file.

### Running the application
There are two ways to get the application running from within WDT:

 * The first is to use Maven to build and run the project:
 1. Run the Maven `install` goal to build and test the project: Right-click **pom.xml** in the `meetings` project, click **Run As… > Maven Build…**, then in the **Goals** field type `install` and click **Run**. The first time you run this goal, it might take a few minutes to download the Liberty dependencies.
 2. Run a Maven build for the `liberty:start-server goal`: Right-click **pom.xml**, click **Run As… > Maven Build**, then in the **Goals** field, type `liberty:start-server` and click **Run**. This starts the server in the background.
 3. Open the application, which is available at `http://localhost:9080/meetings/`.
 4. To stop the server again, run the `liberty:stop-server` build goal.

 * The second way is to right-click the `meetings` project and select **Run As… > Run on Server** but there are a few things to note if you do this. WDT doesn’t automatically add the MicroProfile features as you would expect so you need to manually add those. Also, any changes to the configuration in `src/main/liberty/config` won’t be picked up unless you add an include.

### GitHub
Check out the final code for this project at: https://github.com/WASdev/sample.microprofile.meetingapp
