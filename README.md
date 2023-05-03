Download Link: https://assignmentchef.com/product/solved-cs2200-project5
<br>
<h1>1        Introduction</h1>

This project will expose you to each layer in the network stack and its respective purpose. In particular, you will “upgrade” the transport layer of a simulated network to make it more reliable.

As part of completing this project, you will:

<ul>

 <li>further explore the use of threads in an operating system, especially the network implementation.</li>

 <li>demonstrate how messages are segmented into packets and how they are reassembled.</li>

 <li>understand why a checksum is needed and when it is used.</li>

 <li>understand and implement the stop-and-wait protocol with ACKnowledgements, Negative (NACK) Acknowledgements, and retransmissions.</li>

</ul>

For a description of the Stop-and-Wait Protocol, read Section 13.6.1 in your textbook.

<h1>2        Requirements</h1>

As you work through this project, you will be completing various portions of code in C. There are two files you will need to modify:

<ul>

 <li>c: the main RTP protocol implementation</li>

 <li>h: to add any necessary fields to the rtp connection t struct</li>

</ul>

As you should strive for any programming assignment, we expect quality code. In particular, your code must meet the following requirements:

<ul>

 <li>The code must not generate any compiler warnings</li>

 <li>The code must not print extraneous output by default (i.e. any debug printfs must be disabled by default)</li>

 <li>The code must be reasonably robust and free of memory leaks</li>

</ul>

Code that does not meet these requirements may lose points.

<h1>3        The Protocol Stack</h1>

We have provided you with code that implements the network protocol:

<table width="187">

 <tbody>

  <tr>

   <td width="187"></td>

  </tr>

  <tr>

   <td width="187">Data Link Layer</td>

  </tr>

  <tr>

   <td width="187">Physical Layer</td>

  </tr>

 </tbody>

</table>

client.c rtp.c

network.o

Figure 1: The Protocol Stack

<ul>

 <li>For the purpose of this project, the data link layer and the physical layer are both implemented by the operating system and the underlying network hardware.</li>

 <li>We have implemented our own network layer and provided it to you through the files h and network.c. You should use the provided functions from those files to access the network layer.</li>

 <li>The transport layer uses the services of the network layer to provide a specialized protocol to the application. The transport layer typically provides TCP or UDP services to the application using the IP services provided by the network layer. <strong>For this project, you will be writing your own transport layer.</strong></li>

 <li>The application layer represents the end user application. The application simply makes the appropriate API calls to connect to remote hosts, send and receive messages, and disconnect from remote hosts.</li>

</ul>

<h1>4        Code Walkthrough</h1>

Here, we will briefly describe the code provided for this project. It is important that you study and understand the code given to you. The following diagram displays the interactions between various parts of the code:

The client program takes two arguments. The first argument is the server it should connect to (such as localhost), and the second argument is the port it should connect to (such as 4000). Thus, the client can be run as follows:

$ ./rtp-client localhost 4000

<h2>4.1       High-level Logic</h2>

The client.c program represents the application layer. It uses the services provided by the transport layer (rtp.c). It begins by connecting to the remote host. Look at the rtp connect connection in rtp.c. It simply uses the services provided by the network layer to connect to the remote host. Next, the rtp connect function initializes its rtp connection structure, initializes its send and receive queue, initializes its mutexes, starts its threads, and returns the rtp connection structure.

Next, the client program sends a message to the remote host using rtp send message(). Sending the message could take quite some time if the network connection is slow (imagine sending a 5MB file over a 56k modem). Thus, the rtp send message() message makes a copy of the information to send, places the message into a send queue, and returns so that the application can continue to do other things. A separate thread, the rtp send thread actually sends the data across the network. It waits for a message to be placed into the send queue, then extracts that message from the queue and sends it.

Next, the client program receives a message from the network. What happens if a message isn’t available or the entire message has not yet been received? The rtp receive message() function blocks until a message can be pulled from the receive queue. The rtp recv thread actually receives packets from the network and reassembles the packets into messages. Once it receives a message, it places the message into the receive queue so that rtp receive message can extract it and return it to the application layer.

The client program continues to send and receive messages until it is finished. Last, the client program calls rtp disconnect() to terminate the connection with the remote host. This function changes the state of the connection so that other threads will know that this connection is dead. The rtp disconnect() function then calls net disconnect(), signals the other threads, waits for the threads to finish, empties the queues, frees allocated space, and returns.

<h2>4.2       Packets and Types</h2>

For the purposes of this project, there are five packet types:

<ul>

 <li>DATA – a data packet that contains part of a message in its payload.</li>

 <li>LAST DATA – just like a data packet, but also signifies that it is the last packet in a message.</li>

 <li>ACK – acknowledges the receipt of the last packet</li>

 <li>NACK – a negative acknowledgement stating that the last packet received was corrupted.</li>

 <li>TERM – tells the server to shut down (you don’t need to worry about this one as it’s only used in the provided code).</li>

</ul>

The packet format is defined in network.h. Each packet has a payload, which can be up to MAX PAYLOAD LENGTH bytes, a payload length indicator, type field, and a checksum.

<h1>5        Part I: Segmentation of Data</h1>

When data is sent over a network, the data is chopped up into one or more parts and sent inside packets. A packet contains information that describes the message such as the destination of the data, the source of the data, and the data itself! The data being sent over the network is referred to as the ’payload’. Look in network.h; what other fields does our network packet carry? Think about why each field is needed. How much payload data can we fit into each packet? (Note: as with many things in this project, the packet data structure is simplified).

<strong>(Part A) </strong>Open rtp.c and find the packetize function. Complete this function. Its purpose is to turn a message into an array of packets. It should:

<ol>

 <li>Allocate an array of packets big enough to carry all of the data.</li>

 <li>Populate all the fields of the packet including the payload. Remember, The last packet should be a LAST DATA All other packets should be DATA packets. THIS IS IMPORTANT. The server checks for this, and it will disconnect you if they are not filled in correctly. If you neglect the LAST DATA packet, your program will hang forever waiting for a response from the server, because it is waiting on you forever to send a terminating packet.</li>

 <li>The count variable points to an integer. Update this integer setting it equal to the length of the array you are returning.</li>

 <li>Return the array of packets.</li>

</ol>

Hint: Remember that this is integer division. If length % MAX PAYLOAD LENGTH = 0 this is a special case that should be handled.

There are several other parts of the source code that say FIX ME. The code to be inserted in these parts of the program will simply provide additional functionality but are not necessary at this time. We will return to these parts of the code in Part II

<h1>6        Part II: When Things Go Wrong</h1>

In the stop-and-wait protocol, the sending thread does the following things:

<ol>

 <li>Sends one packet at a time.</li>

 <li>After each packet, wait for an ACK or a NACK to be received.</li>

 <li>If a NACK is received, resend the last packet. Otherwise, send the next packet.</li>

</ol>

The receiving thread should:

<ol>

 <li>Compute the checksum for each packet payload upon arrival.</li>

 <li>If the checksum does not match the checksum reported in the packet header, send a NACK. If it does match, send an ACK.</li>

</ol>

<strong>(Part A) </strong>Open rtp.c and find the checksum function. Complete this function. Simply find the ASCII values of each character in the buffer and return the XOR of all of the values. This is how the server computes the checksum and the server and client must compute the checksum the same way.

<strong>(Part B) </strong>Open rtp.c and find the rtp recv thread function. If the packet is a DATA packet, the payload is added to the current buffer. Modify the implementation so that the data is only added to the buffer if the checksum of the data matches the checksum in the packet header. Next, implement the code that will signal the sending thread that a NACK or ACK has been received. You will also need to determine a way to tell the sending thread whether a negative or positive acknowledgement was received. (Hint: it’s ok to add fields to the rtp connection t data structure).

<strong>(Part C) </strong>Open rtp.c and find the rtp recv thread function. Find the line that says FIX ME: Part II-C. At this point in the function, an entire message has been received. Implement the code such that a new message variable is allocated, and it’s respective fields have been assigned with the buffer’s contents and length. Then, add that message variable to the rtp client’s queue using the provided queue add function. Do this in a thread-safe manner, and signal the recv cond condition variable to let the client know a full message has been received.

<strong>(Part D) </strong>Open rtp.c and find the rtp send thread function. Find the line that says FIX ME: Part II-D. At this point, you should wait to be signaled by the receiving thread that a NACK or ACK has been received. Once notified, take the appropriate action. You should <strong>NOT </strong>call net receive packet in the send thread. The receiving thread is responsible for receiving packets.

<h1>7        Running the Project</h1>

First, you will need to make sure that python 3 is installed on your system to run the server. If you do not have python installed, you can install it with the following commands:

$ sudo apt-get update

$ sudo apt-get install python3.6

Next, to compile all of the code you wrote, use the following command:

$ make

Then to run the server on linux, use the following command:

$ python rtp-server -p [port number] [-c corruption_rate]

<strong>Note: </strong>if the python command above does not work, you may need to replace python with python3.

Your client should work with an UNMODIFIED version of the server in python 3. For example, if you wanted to run a server on port 8080 with a corruption rate of 99%, you would execute the following command:

$ python rtp-server.py -p 8080 -c .99

If you wanted to run a client that would send messages to this server, you would then execute the following command (in a different terminal):

$ ./rtp-client 127.0.0.1 8080

You’ll want to start an instance of the server first, then run the client.

Managing multiple terminal sessions can be a pain, so I would recommend using <a href="https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/">tmux</a> to manage the terminals running your client and the server.

The server will take the client’s messages and make it into 1337 <a href="/cdn-cgi/l/email-protection" class="__cf_email__" data-cfemail="e7c397d4a78c">[email protected]</a> (Leet Speak). The server will be printing out debug statements in order for you to understand what it is doing.


