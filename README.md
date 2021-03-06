# Teaching-HEIGVD-RES-2020-Labo-Orchestra

## Admin

* **You can work in groups of 2 students**.
* It is up to you if you want to fork this repo, or if you prefer to work in a private repo. However, you have to **use exactly the same directory structure for the validation procedure to work**. 
* We expect that you will have more issues and questions than with other labs (because we have a left some questions open on purpose). Please ask your questions on Telegram / Teams, so that everyone in the class can benefit from the discussion.

## Objectives

This lab has 4 objectives:

* The first objective is to **design and implement a simple application protocol on top of UDP**. It will be very similar to the protocol presented during the lecture (where thermometers were publishing temperature events in a multicast group and where a station was listening for these events).

* The second objective is to get familiar with several tools from **the JavaScript ecosystem**. You will implement two simple **Node.js** applications. You will also have to search for and use a couple of **npm modules** (i.e. third-party libraries).

* The third objective is to continue practicing with **Docker**. You will have to create 2 Docker images (they will be very similar to the images presented in class). You will then have to run multiple containers based on these images.

* Last but not least, the fourth objective is to **work with a bit less upfront guidance**, as compared with previous labs. This time, we do not provide a complete webcast to get you started, because we want you to search for information (this is a very important skill that we will increasingly train). Don't worry, we have prepared a fairly detailed list of tasks that will put you on the right track. If you feel a bit overwhelmed at the beginning, make sure to read this document carefully and to find answers to the questions asked in the tables. You will see that the whole thing will become more and more approachable.


## Requirements

In this lab, you will **write 2 small NodeJS applications** and **package them in Docker images**:

* the first app, **Musician**, simulates someone who plays an instrument in an orchestra. When the app is started, it is assigned an instrument (piano, flute, etc.). As long as it is running, every second it will emit a sound (well... simulate the emission of a sound: we are talking about a communication protocol). Of course, the sound depends on the instrument.

* the second app, **Auditor**, simulates someone who listens to the orchestra. This application has two responsibilities. Firstly, it must listen to Musicians and keep track of **active** musicians. A musician is active if it has played a sound during the last 5 seconds. Secondly, it must make this information available to you. Concretely, this means that it should implement a very simple TCP-based protocol.

![image](images/joke.jpg)


### Instruments and sounds

The following table gives you the mapping between instruments and sounds. Please **use exactly the same string values** in your code, so that validation procedures can work.

| Instrument | Sound         |
|------------|---------------|
| `piano`    | `ti-ta-ti`    |
| `trumpet`  | `pouet`       |
| `flute`    | `trulu`       |
| `violin`   | `gzi-gzi`     |
| `drum`     | `boum-boum`   |

### TCP-based protocol to be implemented by the Auditor application

* The auditor should include a TCP server and accept connection requests on port 2205.
* After accepting a connection request, the auditor must send a JSON payload containing the list of <u>active</u> musicians, with the following format (it can be a single line, without indentation):

```
[
  {
  	"uuid" : "aa7d8cb3-a15f-4f06-a0eb-b8feb6244a60",
  	"instrument" : "piano",
  	"activeSince" : "2016-04-27T05:20:50.731Z"
  },
  {
  	"uuid" : "06dbcbeb-c4c8-49ed-ac2a-cd8716cbf2d3",
  	"instrument" : "flute",
  	"activeSince" : "2016-04-27T05:39:03.211Z"
  }
]
```

### What you should be able to do at the end of the lab


You should be able to start an **Auditor** container with the following command:

```
$ docker run -d -p 2205:2205 res/auditor
```

You should be able to connect to your **Auditor** container over TCP and see that there is no active musician.

```
$ telnet IP_ADDRESS_THAT_DEPENDS_ON_YOUR_SETUP 2205
[]
```

You should then be able to start a first **Musician** container with the following command:

```
$ docker run -d res/musician piano
```

After this, you should be able to verify two points. Firstly, if you connect to the TCP interface of your **Auditor** container, you should see that there is now one active musician (you should receive a JSON array with a single element). Secondly, you should be able to use `tcpdump` to monitor the UDP datagrams generated by the **Musician** container.

You should then be able to kill the **Musician** container, wait 5 seconds and connect to the TCP interface of the **Auditor** container. You should see that there is now no active musician (empty array).

You should then be able to start several **Musician** containers with the following commands:

```
$ docker run -d res/musician piano
$ docker run -d res/musician flute
$ docker run -d res/musician flute
$ docker run -d res/musician drum
```
When you connect to the TCP interface of the **Auditor**, you should receive an array of musicians that corresponds to your commands. You should also use `tcpdump` to monitor the UDP trafic in your system.


## Task 1: design the application architecture and protocols

| #  | Topic |
| --- | --- |
|Question | How can we represent the system in an **architecture diagram**, which gives information both about the Docker containers, the communication protocols and the commands? |

![image](images/ArchitectureDiagram.png) 

| #  | Topic |
| --- | --- |
|Question | Who is going to **send UDP datagrams** and **when**? |
| | Musician, continuously as multicast to say it plays sounds on port "x" <br>Musician, continuously as unicast on port "x" where the sounds are |
|Question | Who is going to **listen for UDP datagrams** and what should happen when a datagram is received? |
| | Auditor, to know there is a musician who is playing on port "x", then auditor will listen (if he like the musician) on port "x" to get the musician<br>Auditor, on port "x" to enjoy the music, then he'll chill |
|Question | What **payload** should we put in the UDP datagrams? |
| | instrument, port "x" <br>the sound |
|Question | What **data structures** do we need in the UDP sender and receiver? When will we update these data structures? When will we query these data structures? |
| | The UDP senders needs a stream <br>We will update it every second to send a note<br>We will query this data structure when we receive it's representation on client side.<br><br>The UDP receiver needs a list of musicians, with the timestamp of last heard sound from it. <br>We will update this structure every time we receive a sound from a musician.<br>We qill query those data structures when we want to get all active musician, by comparing the current time and the last time if the musician is active or not.<br> |


## Task 2: implement a "musician" Node.js application

| #  | Topic |
| ---  | --- |
|Question | In a JavaScript program, if we have an object, how can we **serialize it in JSON**? |
| | JSON.stringify({ x: 5, y: 6 })  |
|Question | What is **npm**?  |
| | The node.js package manager, allows to install all kind of packages based on node  |
|Question | What is the `npm install` command and what is the purpose of the `--save` flag?  |
| | `npm install` allows you to install a nodejs package. <br>The `--save` flag adds the package to the dependecies list in package.json  |
|Question | How can we use the `https://www.npmjs.com/` web site?  |
| | We can search for packages before installing them with above command  |
|Question | In JavaScript, how can we **generate a UUID** compliant with RFC4122? |
| | By installing the `uuid` package, then import the package with `import { v1 as uuidv1 } from 'uuid';` then generate the uuid with `uuidv1();`  |
|Question | In Node.js, how can we execute a function on a **periodic** basis? |
| | by using `function intervalFunc() {`<br>`  //something `<br>` } `<br>` setInterval(intervalFunc, 1500);`  |
|Question | In Node.js, how can we **emit UDP datagrams**? |
| | by using the `dgram` package. First import it : `var udp = require('dgram');` <br> then we need to create a scoket with `var server = udp.createSocket('udp4');`<br> Then we send `server.send(msg,info.port,'localhost',function(error){...}` and managin the error in the `...` |
|Question | In Node.js, how can we **access the command line arguments**? |
| | 'use strict'; <br> for (let j = 0; j < process.argv.length; j++) { <br>    console.log(j + ' -> ' + (process.argv[j])); <br> }|


## Task 3: package the "musician" app in a Docker image

| #  | Topic |
| ---  | --- |
|Question | How do we **define and build our own Docker image**?|
| |  with a dockerfile defining the configuration of the dockerized environement, built by docker build command |
|Question | How can we use the `ENTRYPOINT` statement in our Dockerfile?  |
| | by make it pointed to the javascript program  |
|Question | After building our Docker image, how do we use it to **run containers**?  |
| | by running command: docker run -d <name> (-d for background mode) |
|Question | How do we get the list of all **running containers**?  |
| | docker ps  |
|Question | How do we **stop/kill** one running container?  |
| | docker kill  "name" |
|Question | How can we check that our running containers are effectively sending UDP datagrams?  |
| | by checking with wireshark if datagram are sent on ports used by the containers |


## Task 4: implement an "auditor" Node.js application

| #  | Topic |
| ---  | ---  |
|Question | With Node.js, how can we listen for UDP datagrams in a multicast group? |

``` 
const PORT = 6969; 
const MULTICAST_ADDR; 

const dgram = require("dgram"); 
const process = require("process"); 

const socket = dgram.createSocket({ type: "udp4", reuseAddr: true }); 

socket.bind(PORT); 
socket.on("listening", function() { 
  socket.addMembership(MULTICAST_ADDR); 
  setInterval(sendMessage, 2500); 
  const address = socket.address(); 
  console.log( 
    UDP socket listening on ${address.address}:${address.port} pid: ${
      process.pid
    } 
  ); 
}); 
```


| #  | Topic |
| ---  | --- |
|Question | How can we use the `Map` built-in object introduced in ECMAScript 6 to implement a **dictionary**?  |
| | |
```
var myMap = new Map();
myMap.set("key", "value");
myMap.get("key"); //return "value"
```

| #  | Topic |
| --- | --- |
|Question | How can we use the `Moment.js` npm module to help us with **date manipulations** and formatting?  |
| | |
```
//add days
moment().add(1, 'days').calendar();  
//substract days
moment().subtract(1, 'days').calendar(); 
//format
moment().format('MMMM Do YYYY, h:mm:ss a');
```

| #  | Topic |
| --- | --- |
|Question | When and how do we **get rid of inactive players**?  |
| | We get rid of inactive players as soon as they are inactive. To detect that musician are inactive, we compare the timestamps between the saved value and the current time stamp, if it's greater then 5 second, we get rid of the musician |
|Question | How do I implement a **simple TCP server** in Node.js?  |
| | using net.Server library| 
```
var net = require('net');
var server = net.createServer(function(socket) {
	socket.write('Echo server\r\n');
	socket.pipe(socket);
});
server.listen(1337, '127.0.0.1');
```



## Task 5: package the "auditor" app in a Docker image

| #  | Topic |
| ---  | --- |
|Question | How do we validate that the whole system works, once we have built our Docker image? |
| | By running the validate.sh script located in the top-level directory.<br>When this works, we then create a auditor, require active musician from it with tcp query. then add some musician, let them play and ask for active musician. <br>Then stops the musicians, wait for 5 seconds and ask for active musician |


## Constraints

Please be careful to adhere to the specifications in this document, and in particular

* the Docker image names
* the names of instruments and their sounds
* the TCP PORT number

Also, we have prepared two directories, where you should place your two `Dockerfile` with their dependent files.

Have a look at the `validate.sh` script located in the top-level directory. This script automates part of the validation process for your implementation (it will gradually be expanded with additional operations and assertions). As soon as you start creating your Docker images (i.e. creating your Dockerfiles), you should try to run it.
