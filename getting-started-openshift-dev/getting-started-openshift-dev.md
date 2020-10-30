# Getting Started with OpenShift for Developers

## Goal

**Learn how to use the OpenShift Container Platform to build and deploy an application with a data backend and a web frontend.**

## Concepts



* OpenShift Web Console and Perspectives
* OpenShift <code>oc</code> command line tool
* Building applications from source on OpenShift
* OpenShift REST API
* Public URLs and OpenShift Routes

## Use case


Be able to provide a great experience for both Developers and System Administrators to develop, deploy, and run containerized applications using OpenShift. Developers should love using OpenShift because it enables them to take advantage of both containerized applications and orchestration without having to know the details. Developers are free to focus on their code instead of spending time writing Dockerfiles and running docker builds.


# Step 1 - Exploring The Command Line


## Command Line Interface (CLI)


The OpenShift CLI is accessed using the command _oc_. From here, you can administrate the entire OpenShift cluster and deploy new applications.


The CLI exposes the underlying Kubernetes orchestration system with the enhancements made by OpenShift. Users familiar with Kubernetes will be able to adapt to OpenShift quickly. _oc_ provides all of the functionality of _kubectl_, along with additional functionality to make it easier to work with OpenShift. The CLI is ideal in situations where you are:


1) Working directly with project source code
2) ScriptinOpenShiftg OpenShift operations
3) Restricted by bandwidth resources and cannot use the web console


In this tutorial, we're not focusing on the OpenShift CLI, but we want you to be aware of it in case you prefer using the command line. You can check out our other courses that go into the use of the CLI in more depth. Now, we're just going to practice logging in so you can get some experience with how the CLI works.


## Exercise: Logging in with the CLI


#### Let's get started by logging in. Your task is to enter the following into the console:


```
oc login
```



**When prompted, enter the following username and password:**


Username: <code>developer</code></strong>


Password: <code>developer</code></strong>


Next, you can check if it was successful:

```
oc whoami
```

<code>oc whoami</code> should return a response of:

```

developer
```

**That's it!**

In the next step, we'll get started with creating your first project using the web console.


# Step 2 - Exploring The Web Console 

This section focuses on using the web console.


## Exercise: Logging in with the Web Console

To begin, click on the **Console** tab on your screen. This will open the web console on your browser.

You should see a **Red Hat OpenShift Container Platform** window with **Username** and **Password** forms as shown below:


![alt_text](images/image21.png "image_tooltip")


For this scenario, log in by entering the following:

**Username:** `developer`

**Password:** `developer`

After logging in to the web console, you'll be on a _Projects_ page.


## **What is a project? Why does it matter?**

OpenShift is often referred to as a container application platform in that it is a platform designed for the development and deployment of applications in containers.

To group your application, we use projects. The reason for having a project to contain your application is to allow for controlled access and quotas for developers or teams.

More technically, it's a visualization of the Kubernetes namespace based on the developer access controls.


## **Exercise: Creating a Project**

Click the blue **Create Project** button.

You should now see a page for creating your first project in the web console. Fill in the _Name_ field as `myproject`.


![alt_text](images/image20.png "image_tooltip")


The rest of the form is optional and up to you to fill in or ignore. Click _Create_ to continue.

After your project is created, you will see some basic information about your project.


## **Exercise: Explore the Administrator and Developer Perspectives**

Notice the navigation menu on the left. When you first log in, you'll typically be in the _Administrator Perspective_. If you are not in the _Administrator Perspective_, click the perspective toggle and switch from **Developer** to **Administrator**.



![alt_text](images/image12.png "image_tooltip")


You're now in the _Administrator Perspective_, where you'll find **Operators**, **Workloads**, **Networking**, **Storage**, **Builds**, and **Administration** menus in the navigation.

Take a quick look around these, clicking on a few of the menus to see more options.

Now, toggle to the _Developer Perspective_. We will spend most of our time in this tutorial in the _Developer Perspective_. The first thing you'll see is the _Topology_ view. Right now it is empty, and lists several different ways to add content to your project. Once you have an application deployed, it will be visualized here in _Topology_ view.


# Step 3 - Deploying a Docker Image


In this section, you are going to deploy the front end component of an application called parksmap. The web application will display an interactive map, which will be used to display the location of major national parks from all over the world.


## Exercise: Deploying Your First Image


The simplest way to deploy an application in OpenShift is to take an existing container image and run it. We are going to use the OpenShift web console to do this, so ensure you have the OpenShift web console open with the _Developer Perspective_ active and that you are in the project called <code>myproject</code>.</strong>


The OpenShift web console provides various options to deploy an application to a project. For this section, we are going to use the _Container Image_ method. As the project is empty at this point, the _Topology_ view should display the following options: _From Git_, _Container Image_, _From Catalog_, _From Dockerfile_, _YAML_, and _Database_.


**Choose the Container Image option.**


![alt_text](images/image16.png "image_tooltip")



In the future, to get back to this menu of ways to add content to your project, you can click _+Add_ in the left navigation.


Within the _Deploy Image_ page, enter the following for _Image name from external registry_:


```
docker.io/openshiftroadshow/parksmap-katacoda:1.2.0
```



Press tab or click outside of the text box to validate the image:



![alt_text](images/image13.png "image_tooltip")

he _Application Name_ field will be populated with <code>parksmap-katacoda-app</code> and the <em>Name</em> field with <code>parksmap-katacoda</code>. This name will be what is used for your application and the various components created that relate to it. Leave this as the generated value as steps given in the upcoming sections will use this name.


By default, creating a deployment using the _Container Image_ method will also create a Route for your application. A Route makes your application available at a publicly accessible URL.



![alt_text](images/image6.png "image_tooltip")



Normally, you would keep this box checked, since it's very convenient to have the Route created for you. For the purposes of learning, un-check the box. We'll learn more about Routes later in the tutorial, and we'll create the Route ourselves then.


You are ready to deploy the existing container image. Click the blue _Create_ button at the bottom of the screen. This should bring you back to the _Topology_ view, where you'll see a visual representation of the application you just deployed. As the image deployment progresses, you'll see the ring around the <code>parksmap-katacoda</code> deployment progress from white to light blue to blue.


![alt_text](images/image23.png "image_tooltip")



These are the only steps you need to run to get a "vanilla" container image deployed on OpenShift. This should work with any container image that follows best practices, such as defining the port any service is exposed on, not needing to run specifically as the _root user_ or other dedicated user, and which embeds a default command for running the application.


# Step 4 - Scaling Your Application


Let's scale our application up to 2 instances of the pods. You can do this by clicking inside the circle for the <code>parksmap-katacoda</code> application from <em>Topology</em> view to open the side panel. In the side panel, click the <em>Details</em> tab, and then click the "up" arrow next to the <em>Pod</em> in side panel.


![alt_text](images/image3.png "image_tooltip")


To verify that we changed the number of replicas, click the _Resources_ tab in the side panel. You should see a list with your pods similar to the image below:


![alt_text](images/image18.png "image_tooltip")

You can see that we now have 2 replicas.


verall, that's how simple it is to scale an application (_Pods_ in a _Service_). Application scaling can happen extremely quickly because OpenShift is just launching new instances of an existing image, especially if that image is already cached on the node.


### Application "Self Healing"


OpenShift's _Deployments_ are constantly monitoring to see that the desired number of _Pods_ is actually running. Therefore, if the actual state ever deviates from the desired state (i.e., 2 pods running), OpenShift will work to fix the situation.


Since we have two _Pods_ running right now, let's see what happens if we "accidentally" kill one.


On the _Resources_ tab where you viewed the list of pods after scaling to 2 replicas, open one of the pods by clicking its name in the list.


In the top right corner of the page, there is an _Actions_ drop down menu. Click it and select _Delete Pod_.


![alt_text](images/image9.png "image_tooltip")

After you click _Delete Pod_, click _Delete_ in the confirmation dialog. You will be taken to a page listing pods, however, this time, there are three pods. Note that on smaller screens you may not see all of these columns.


![alt_text](images/image2.png "image_tooltip")



The pod that we deleted is terminating (i.e., it is being cleaned up). A new pod was created because OpenShift will always make sure that, if one pod dies, there is going to be new pod created to fill its place.


### Exercise: Scale Down


Before we continue, go ahead and scale your application down to a single instance. Click _Topology_ to return to the _Topology_ view, then click <code>parksmap-katacoda</code> and on the <em>Overview</em> tab, click the down arrow to scale back down to one instance.


# Step 5 - Routing HTTP Requests


_Services_ provide internal abstraction and load balancing within an OpenShift environment, but sometimes clients (users, systems, devices, etc.) outside of OpenShift need to access an application. The way that external clients are able to access applications running in OpenShift is through the OpenShift routing layer. The resource object which controls this is a _Route_.


The default OpenShift router (HAProxy) uses the HTTP header of the incoming request to determine where to proxy the connection. You can optionally define security, such as TLS, for the _Route_. If you want your _Services_, and, by extension, your _Pods_, to be accessible to the outside world, you need to create a _Route_.

As we mentioned earlier in the tutorial, the _Container Image_ method of deploying an application will create a _Route_ for you by default. Since we un-checked that option, we will manually create a _Route_ now.


## Exercise: Creating a Route


Fortunately, creating a _Route_ is a pretty straight-forward process. First, go to the _Administrator Perspective_ by switching to _Administrator_ in the _Developer_ drop down menu. Ensure that your <code>myproject</code> project is selected from the projects list. Next, click <em>Networking</em> and then <em>Routes</em> in the left navigation menu.


**Click the blue _Create Route_ button.


![alt_text](images/image7.png "image_tooltip")



Enter <code>parksmap-katacoda</code> for the <em>Route</em> Name, select <code>parksmap-katacoda</code> for the <em>Service</em>, and <code>8080</code> for the Target Port. Leave all the other settings as-is.

![alt_text](images/image8.png "image_tooltip")

Once you click _Create_, the _Route_ will be created and displayed in the _Route Details_ page.


![alt_text](images/image17.png "image_tooltip")



You can also view your _Route_ in the _Developer Perspective_. Toggle back to the _Developer Perspective_ now, and go to _Topology_ view. On the <code>parksmap-katacoda</code> visualization you should now see an icon in the top right corner of the circle. This represents the <em>Route</em>, and if you click it, it will open the URL in your browser.

![alt_text](images/image19.png "image_tooltip")

Once you've clicked the _Route_ icon, you should see this in your browser:

![alt_text](images/image15.png "image_tooltip")



# Step 6 - Building From Source Code

In this section, you are going to deploy a backend service for the ParksMap application. This backend service will provide data, via a REST service API, on major national parks from all over the world. The ParksMap front end web application will query this data and display it on an interactive map in your web browser.


**Background: Source-to-Image (S2I)**

In a previous section, you learned how to deploy an application (the ParksMap front end) from a pre-existing container image. Here you will learn how to deploy an application direct from source code hosted in a remote Git repository. This will be done using the [Source-to-Image (S2I)](https://github.com/openshift/source-to-image) tool.

The documentation for S2I describes itself in the following way:

Source-to-image (S2I) is a tool for building reproducible container images. S2I produces ready-to-run images by injecting source code into a container image and assembling a new container image which incorporates the builder image and built source. The result is then ready to use with docker run. S2I supports incremental builds which re-use previously downloaded dependencies, previously built artifacts, etc.

OpenShift is S2I-enabled and can use S2I as one of its build mechanisms (in addition to building container images from Dockerfiles and "custom" builds).

A full discussion of S2I is beyond the scope of this tutorial. More information about S2I can be found in the [OpenShift S2I documentation](https://docs.openshift.com/container-platform/latest/builds/understanding-image-builds.html#build-strategy-s2i_understanding-image-builds) and the [GitHub project respository for S2I](https://github.com/openshift/source-to-image).

The only key concept you need to remember about S2I is that it handles the process of building your application container image for you from your source code.


## Exercise: Deploying the application code

The backend service that you will be deploying in this section is called `nationalparks-katacoda`. This is a Python application that will return map coordinates of major national parks from all over the world as JSON via a REST service API. The source code repository for the application can be found on GitHub at:



*   [https://github.com/openshift-roadshow/nationalparks-katacoda](https://github.com/openshift-roadshow/nationalparks-katacoda)

To deploy the application you are going to use the **+Add** option in the left navigation menu of the _Developer Perspective_, so ensure you have the OpenShift web console open and that you are in the project called `myproject`. Click **+Add**. This time, rather than using _Container Image_, choose _From Catalog_, which will take you to the following page:


![alt_text](images/image5.png "image_tooltip")


If you don't see any items, then uncheck the **Operator Backed** checkbox. Under the **Languages** section, select Python in the list of supported languages. When presented with the options of _Django + Postgres SQL_, _Django + Postgres SQL (Ephemeral)_, and _Python_, select the _Python_ option and click on _Create Application_.

![alt_text](images/image11.png "image_tooltip")

For the _Git Repo URL_ use:

```
https://github.com/openshift-roadshow/nationalparks-katacoda
```

![alt_text](images/image14.png "image_tooltip")


Once you've entered that, click outside of the text entry field, and then you should see the _Name_ of the application show up as `nationalparks-katacoda`. The _Name_ needs to be `nationalparks-katacoda` as the front end for the ParksMap application is expecting the backend service to use that name.

Leave all other options as-is.

Click on _Create_ at the bottom right corner of the screen and you will return to the _Topology_ view. Click on the circle for the `nationalparks-katacoda` application and then the _Resources_ tab in the side panel. In the _Builds_ section, you should see your build running.


![alt_text](images/image22.png "image_tooltip")


This is the step where S2I is run on the application source code from the Git repository to create the image which will then be run. Click on the _View Logs_ link for the build and you can follow along as the S2I builder for Python downloads all the Python packages required to run the application, prepares the application, and creates the image.


![alt_text](images/image4.png "image_tooltip")


Head back to _Topology view_ when the build completes to see the image being deployed and the application being started up. The build is complete when you see the following in the build logs: `Push successful`.


![alt_text](images/image1.png "image_tooltip")


The green check mark in the bottom left of the `nationalparks-katacoda` component visualization indicates that the build has completed. Once the ring turns from light blue to blue, the backend `nationalparks-katacoda` service is deployed.

Now, return to the ParksMap front end application in your browser, and you should now be able to see the locations of the national parks displayed. If you don't still have the application open in your browser, go to _Topology_ view and click the icon at the top right of the circle for the `parksmap-katacoda` application to open the URL in your browser.


![alt_text](images/image10.png "image_tooltip")

Congratulations! You just finished learning the basics of how to get started with the OpenShift Container Platform.

Now that you've completed this tutorial, click _Continue_ for more resources and tools to help you learn more about OpenShift.
