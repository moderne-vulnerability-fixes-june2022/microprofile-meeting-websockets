# Part 4: MicroProfile Meeting Application - Using WebSockets and CDI events

## Overview
In this lab we will cover adding WebSockets and using CDI Events to integrate a WebSocket and the CDI beans so that the server can notify clients about changes

If you have eagle eyes, you might have noticed with the application that the redirect only works when a meeting is started, not when it is already running. In order to make the join action work, the browser needs to find out when the meeting has changed. This can be done by polling the server but that can be expensive. Instead, a WebSocket can be used to allow the server to notify the client. This reduces the number of requests to the server and provides prompt updates to the client.

Adapted from the blog post: [Writing a simple MicroProfile application: Using WebSockets and CDI events](https://developer.ibm.com/wasdev/docs/simple-microprofile-application-using-websockets-cdi-events/)

## Prerequisites
 * Completed [Part 3: MicroProfile Meeting Application - Using Java EE Concurrency](https://github.com/IBM/microprofile-meeting-concurrency)
 * [Eclipse Java EE IDE for Web Developers](http://www.eclipse.org/downloads/)
 * IBM Websphere Application Liberty Developer Tools (WDT)
   1. Start Eclipse
   2. Launch the Eclipse Marketplace: **Help** -> **Eclipse Marketplace**
   3. Search for **IBM Websphere Application Liberty Developer Tools**, and click **Install** with the defaults configuration selected
 * [Git](https://git-scm.com/downloads)
 * [Bluemix Account](https://www.bluemix.net)
 * [Bluemix CLI](https://clis.ng.bluemix.net/ui/home.html)
 
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

### Step 2. Deploy MongoDB Instance in Bluemix
1. Log in to your [Bluemix Account](https://www.bluemix.net)
2. Deploy a [Compose for MongoDB](https://console.bluemix.net/catalog/services/compose-for-mongodb) instance.

### Step 3. Updating the application to compile against the WebSocket API

### Step 4. Create a CDI qualifier

### Step 5. Creating a CDI event object

### Step 6. Sending an event when a meeting starts

### Step 7. Creating the WebSocket

### Step 8. Configuring Liberty to run WebSockets

### Running the application
There are two ways to get the application running from within WDT:

 * The first is to use Maven to build and run the project:
 1. Run the Maven `install` goal to build and test the project: Right-click **pom.xml** in the `meetings` project, click **Run As… > Maven Build…**, then in the **Goals** field type `install` and click **Run**. The first time you run this goal, it might take a few minutes to download the Liberty dependencies.
 2. Run a Maven build for the `liberty:start-server goal`: Right-click **pom.xml**, click **Run As… > Maven Build**, then in the **Goals** field, type `liberty:start-server` and click **Run**. This starts the server in the background.
 3. Open the application, which is available at `http://localhost:9080/meetings/`.
 4. To stop the server again, run the `liberty:stop-server` build goal.

 * The second way is to right-click the `meetings` project and select **Run As… > Run on Server** but there are a few things to note if you do this. WDT doesn’t automatically add the MicroProfile features as you would expect so you need to manually add those. Also, any changes to the configuration in `src/main/liberty/config` won’t be picked up unless you add an include.

Find out more about [MicroProfile and WebSphere Liberty](https://developer.ibm.com/wasdev/docs/microprofile/).
