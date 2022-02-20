## Testbed experiment

### Background

In this experiment, we're going to use a testbed experiment to measure the average queue length in the following scenario:

![](mm1-scenario-1.svg)

We know that the mean number of packets in this system is given by ρ/(1−ρ), where ρ=λ/μ and λ and μ are the rate at which packets arrive at the queue (in packets/second) and the rate at which packets may be served by the queue (in packets/second). (In our experiment, the distribution of μ comes from variations in the size of packets arriving at the queue, which has a constant service rate in bits/second - μ is computed by dividing the queue service rate in bits/second by the average packet size in bits.)

From the mean number of packets in the system, we can compute the mean number of packets in the queue by substracting the mean number of packets in service: ρ/(1−ρ)−ρ=ρ^2/(1−ρ). This is the result we will try to confirm with a testbed experiment.

### Set up resources

To start, create a new slice on the GENI portal. Create a client-router-server topology like the one shown here, and assign IP addresses to each host (you can use the Auto IP button in the GENI portal, or manually assign addresses).

![](https://witestlab.poly.edu/blog/content/images/2021/03/line-topo.png)

Once your resources come up, log in to each node. Install `iperf3`, which we'll use to measure the link capacities:

```
sudo apt update
sudo apt -y install iperf3
```

Verify that traffic between client and server goes via the router, and find out the properties (latency, capacity) of the individual links: the link between client and router, and the link between router and server. To estimate the latency of a link, you can use `ping` between two endpoints of a link, and to estimate the capacity of a link, you can use `iperf3`. 

To generate traffic with exponentially distributed interarrival times and exponentially distributed packet sizes, we will use the D-ITG traffic generator. The user manual for this software is available [here](http://traffic.comics.unina.it/software/ITG/manual/).

You will need to install D-ITG, a software traffic generator, on your client and server nodes. Log in to each and run

```
sudo apt update # refresh local information about software repositories
sudo apt -y install d-itg # install D-ITG software package
```

Now, the D-ITG tools should be installed on both the client and server nodes.

Also, install `tshark` on the router node:

```
sudo apt update # refresh local information about software repositories
sudo apt install tshark
```

Finally, run

```
ifconfig
```

on all three nodes. Use the IP addresses assigned to each interface to identify the names of the interfaces (e.g. "eth1", "eth2") on the router

* through which traffic from the client enters the router. This will be the interface whose IP address is in the same subnet as the client's experiment interface.
* through which traffic leaves the router towards the server. This will be the interface whose IP address is in the same subnet as the server's experiment interface.

### Practice using D-ITG

D-ITG comes with a suite of utilities for generating traffic, receiving traffic, parsing log files, and automating experiments. We will use learn how to use three of the D-ITG utilities in: the sender (traffic generator), the receiver (traffic sink), and the log file decoder. We will run the sender on the client node, the receiver on the server node, and the log file decoder on both the client and server node.

Try it now with the following command on the receiver (server) node:

```
ITGRecv
```

Leave that running, and on the sender (client) nodes, run:

```
ITGSend -a server -l sender.log -x receiver.log -E 200 -e 512 -T UDP -t 240000
```

This will send UDP traffic to the server with a packet interarrival time that is exponentially distributed with mean 200 packets/second, packet size that is exponentially distributed with mean 512 bytes. It will run for 4 minutes (240000 ms), and will save the log files at the client node and server node with file names "sender.log" and "receiver.log" respectively.

Wait for the sender to finish and then stop the receiver with Ctrl+C. Then, open the log on the receiver side with

```
ITGDec receiver.log
```

and on the sender side with

```
ITGDec sender.log
```


Here you can see some basic aggregate information about the generated traffic flow(s). More detailed information is available from the log file using various arguments; see the documentation for details. (Note that the delay information reported is not reliable, since the clocks on the two nodes are not synchronized. You may even see a negative delay reported.)

In general, the basic usage of the sender will be as follows:

```
ITGSend -l SENDER-LOG -x RECEIVER-LOG -E MEAN-ARRIVAL-RATE -e MEAN-PACKET-SIZE  -t EXPERIMENT-DURATION -T UDP
```

where SENDER-LOG is the name to use for the log file at the sender side, RECEIVER-LOG is the name to use for the log file at the receiver side, MEAN-ARRIVAL-RATE is the mean rate at which to send traffic in packets per second (the -E indicates that the arrival rate will be exponentially distributed), MEAN-PACKET-SIZE is the mean size of the data payload of the packet in bytes (the -e indicates that the packet size will be exponentially distributed), and EXPERIMENT-DURATION is the length of the experiment in milliseconds (the default is 10 seconds). You can choose MEAN-ARRIVAL-RATE and MEAN-PACKET-SIZE to create a traffic pattern with any λ and any μ that you want. (For our experiments today, we will only vary λ and will keep the mean packet size constant.)

> **Note**: As you run your experiments, if you become disconnected from your server node and have to reconnect, you may encounter this error when you try to run the D-ITG receiver:
>
> ```
> ** ERROR_TERMINATE **
> Function main aborted caused by general parser 
> ** Cannot bind a socket on port 9000 for signaling ** 
> Finish requested caused by errors! 
> ```
>
> This indicates that a D-ITG receiver is already running in the background. To kill it (so that you can start a new one), run `sudo killall ITGRecv`.

Open another terminal and log in to the router node. On this terminal, we will use `tshark` to capture incoming traffic on the experiment interface.


```
sudo tshark -i eth1 -f "udp and port 8999" -T fields -e frame.time_delta_displayed -e frame.len -E separator=, | tee output.csv
```


Here,

* the `-i` flag specifies which network interface to listen on (in your experiment, use the experiment/data interface through which traffic enters the router from the client - either eth1 or eth2),
* we use `-f "udp and port 8999"` to filter out traffic that is not generated by D-ITG,
* we specify a list of fields to print with `-T` fields:
  * `frame.time_delta_displayed` is the time delta between displayed frames
  * `frame.len` is the size of the frame, in bytes
* and we use a comma to separate fields in the output, with `-E separator=,`

We also multiplex the output using `tee`, so it appears on the terminal but is also saved in a file called "output.csv".

Re-run the D-ITG sender and receiver on your client and server nodes. After these have finished, stop the `tshark` with Ctrl+C. Then, you can transfer the "output.csv" file to your laptop (e.g. with scp) and open it in Excel, MATLAB, or any other data analysis tool.

### Learn basic usage of token bucket filter queue in `tc`

The traffic source and sink applications will run on the client and server nodes. On the router node, we will deploy a queue, using the Linux Traffic Control (`tc`) utility to manipulate queue settings. 

For this experiment, we will use a kind of token bucket filter queue, documented [here](https://linux.die.net/man/8/tc-htb). This queue shapes traffic using tokens, which arrive at a steady rate into a token bucket (which has a maximum capacity equal to the "burst size" of the queue). Meanwhile, traffic arrives at the token bucket from a separate queue. To leave the queue (and be transmitted), a packet must consume a number of packets roughly equal to its size in bytes. If there is not a sufficient number of tokens available when a packet arrives at the head of the queue, it must wait for more tokens to be generated. The queue, like the token bucket, has a finite length; if a packet arriving at the queue finds it full, the packet is dropped.

The model of the token bucket filter looks something like this (image via [unix.se](http://unix.stackexchange.com/a/100797)):

![](http://i.stack.imgur.com/200us.png)


Try setting the queue to serve traffic at a rate of 1Mbps by running the following command on the router (in the following commands, change `eth2` to the name of the router interface "facing" the server):

```
# Delete any existing queues
# don't worry about error message on this command
sudo tc qdisc del dev eth2 root  

# Create an htb queue
sudo tc qdisc replace dev eth2 root handle 1: htb default 3  

# Set up rate limiting
sudo tc class add dev eth2 parent 1: classid 1:3 htb rate 1mbit  

# Set FIFO queue capacity in bytes
sudo tc qdisc add dev eth2 parent 1:3 bfifo limit 500mbit  
```

Here, I have specified the interface on which to queue outgoing traffic (use the interface through which traffic leaves the router towards the server - in my case, `eth2`), the rate at which the queue should operate (1 Mbps), and the maximum size of the queue (500 megabits, i.e. "very large"). 

Use `iperf3` again to verify the new bottleneck link rate.


The `tc` tool includes a command to show the current state of a queue. Try running it, specifying as the last interface the name of the network interface you want to monitor (e.g. `eth1` in this example):


```
tc -p -s -d qdisc show dev eth2
```


The output of this command may look something like this:

```
qdisc htb 1: root refcnt 2 r2q 10 default 3 direct_packets_stat 8995 ver 3.17 direct_qlen 1000
 Sent 159261600 bytes 1043 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 119606b 58p requeues 0
qdisc bfifo 8002: parent 1:3 limit 64000Kb
 Sent 31322502 bytes 849 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 119606b 58p requeues 0
```

This output shows that the active queuing discipline (`qdisc`) on my interface is Hierarchy Token Bucket (`htb`). I can also see the current queue statistics. The most relevant of these (for our purposes) is the "backlog" information, which tells us the number of bytes and the number of packets currently in the queue. Also important is the "dropped" value, which tells you whether packets were dropped by the queue, and how many.

For your convenience, I have written a simple bash script that you can run on the router node to watch the queue occupancy:

```
#!/bin/bash

if [ "$#" -ne 3 ]; then
    echo "Usage: $0 interface duration interval"
    echo "where 'interface' is the name of the interface on which the queue"
    echo "running (e.g., eth2), 'duration' is the total runtime (in seconds),"
    echo "and 'interval' is the time between measurements (in seconds)"
    exit 1
fi

interface=$1
duration=$2
interval=$3

# Run for the number of seconds specified by the "duration" argument
end=$((SECONDS+$duration))

while [ $SECONDS -lt $end ]
do
        # print timestamp at the beginning of each line;
        # useful for data analysis. (-n argument says not to
        # create a newline after printing timestamp.)
        echo -n "$(date +%s.%N) "
        # Show current queue information on the specified
        # interface, all on one line.
        echo $(tc -p -s -d qdisc show dev $interface)
        # sleep for the specified interval
        sleep $interval
done
```

Save the contents of this script in a file on your router node, e.g. as `queuemonitor.sh`, and then make it executable (e.g. `chmod a+x queuemonitor.sh`).

To use this script, run

```
./queuemonitor.sh INTERFACE DURATION INTERVAL
```


where INTERFACE is the name of the network interface on which the queue is running, DURATION is the total time for which to monitor the queue, and INTERVAL specifies how long to sleep in between measurements.

Try running it now, on the router node, while you repeat your D-ITG experiment. You may find it useful to simultaneously watch the output on stdout and redirect it to a file at the same time with `tee`, e.g.

```
./queuemonitor.sh eth2 240 0.1 | tee router.txt
```

(make sure to specify the appropriate interface name for your experiment.) Each line of output may look something like this:

```
1616593533.619288051 qdisc htb 1: root refcnt 2 r2q 10 default 3 direct_packets_stat 8995 ver 3.17 direct_qlen 1000 Sent 195768011 bytes 3796 pkt (dropped 131, overlimits 54406 requeues 0) backlog 1651840b 633p requeues 0 qdisc bfifo 8002: parent 1:3 limit 64000Kb Sent 67828913 bytes 3602 pkt (dropped 0, overlimits 0 requeues 0) backlog 1651840b 633p requeues 0
```


After an experiment is over, you can process this file with your data analysis tool of choice. For your convenience, if you have redirected the output to `router.txt`, you can retrieve the average queue occupancy (in packets) with:

```
cat router.txt | sed 's/\p / /g' | awk  '{ sum += $54 } END { if (NR > 0) print sum / NR }'
```

This prints the contents of the file to the terminal, then uses `sed` to replace instances of "p" with a blank space " " (since the value we are interested in is in the form "2p"). Finally, it uses `awk` to find the mean value of the 54th column.

To remove a traffic shaping queue, you can run

```
sudo tc qdisc del dev eth2 root
```

(substituting the name of the interface that you want to modify). This will remove any non-default queues you have added, and restore the default queue on that interface.


### Validating the experiment

In our experiment, we assume that:

* The packet sizes are exponentially distributed with mean size 512 bytes, so that the mean service rate at the router is exponentially distributed with mean 244.1 packets/second.
* The traffic is Poisson with λ = 200 packets/second (at least, for the initial experiment that we ran on each platform).
* The queue capacity is effectively infinite (i.e. no packets are dropped).


Now, we will try and validate how well these assumptions are realized in our testbed experiment.

Traffic generators may not perfectly represent the model they are supposed to - traffic generators (including D-ITG) do not always achieve exactly what they have promised. To understand whether or not this can be expected to affect our experiment, we will have to validate the performance of D-ITG in our specific experiment scenario. Specifically, we want to see how well D-ITG mimics an exponential packet interarrival time and exponential packet size distribution under the circumstances we are running it in: with data rates of approximately 200 packets/second, and an average packet size of 512 bytes.

You may have noticed from the output of D-ITG that you did not get the exact rate of packets/second that you requested. Similarly, if you divide the reported number of bytes sent by the reported number of packets sent, you may find that it isn't exactly the average packet size you asked for. In fact, the situation may be even worse than that!

After going through the steps in the previous sections, you should have four records of the mean packet size from an M/M/1 experiment: the two D-ITG log files, the tshark output, and the queue monitor output (where you can compute the difference in the number of bytes sent between the last and first lines of the experiment, compute the different in the number of packets sent, and divide to get the mean packet size). 

**Lab report**: Create a table listing the four records in the first column, and the mean packet size suggested by this record in the second column. Do those four numbers agree?

You may observe that the mean packet size measured by the router is larger than expected. This is because the size that we pass as an argument to D-ITG is only for the data payload, but the total size of the frame as it transits through the router will also include:

* an 8-byte UDP header
* a 20-byte UDP header
* a 14-byte Ethernet header

Thus the mean packet size seen by the router is slightly more than 512 B, and the mean service rate will therefore be a bit lower than anticipated. To have D-ITG generate packets with mean size 512 B as seen by the router, we should actually tell it to use a mean data payload size of 470 B - another 42 B of header will be added to each packet by the OS.

That's not the only difference between our model and the realization of it, though. From the `tshark` output, find out the frequency of each packet size in the trace (e.g. number of packets of size 1, number of packets of size 2, ... up to the number of packets of size 2500). 

Then, create a plot showing the distribution of packets sizes compared to the expected distribution, as follows:

* put "Packet size in bytes" on the x-axis 
* put "Number of packets of this size observed" on the y-axis 
* Use a log10 scale for the y-axis, and a range of 0 to 2500 B on the x-axis 
* On top of this plot, draw a line showing the number of packets of each size that you expect to observe if packet sizes are exponentially distributed with mean 512 B.


If our packets were generated according to an exponential distribution with mean 512 B, we would expect to see something like this:

![](packetsize-distribution-ideal.svg)



In your output,

* Do you observe a much higher number of ~1514-byte frames than expected?
* Do you observe any frames larger than ~1514 bytes?
* Do you observe more frames in the range 62-1514 bytes than expected?
* Do you observe a much higher number of ~62-byte frames than expected?
* Do you observe any frames smaller than 62 bytes?

You are likely to see the effects of fragmentation in your plot. The network interfaces in our experiment have an MTU of 1500 B; this is the maximum size IP packet it will transmit. Any packet larger than this will be fragmented. Thus, large packets generated by D-ITG will be transmitted as a 1514 B frame (1500 B + 14 B Ethernet header) followed by one or more fragments. We therefore see many more 1514 B frames than expected, and no frames larger than 1514 B. We also see more frames than expected between 62-1514 bytes, because the fragments will be counted in this range.

We also note that the minimum size of an Ethernet frame is 64 bytes. (In our experiment, we see frames as small as 62 bytes because 2 bytes of padding will be added to the Ethernet header by the interface when it is transmitted, but after it is seen by `tshark`.) Smaller packets will be padded to 62 bytes. We therefore see many more 62 B frames than expected, and no frames that are smaller.

Because of these effects - header overhead, MTU, and minimum frame size - the actual mean packet size in the testbed experiment will not be 512 B. We can give D-ITG a mean payload size of 470 B, but even then we won't get a mean frame size of 512 B. This is partly because small frames will be padded to 62 B and also because, if a packet is fragmented, then a 42 B header is appended to each of its fragments. (And of course, we also won't get a perfect exponential distribution of packet sizes.)

**Lab report**: Show the plot of actual vs. expected packet size. Comment on your observations.


We will also consider the packet interarrival time for our testbed experiment. We expect a mean interarrival time of 0.005 second (1/200 packets/second). Compute the observed mean interarrival time using the CSV file produced by tshark - how close is it to 0.005 seconds? Also create a plot showing the expected and observed distribution. From the tshark output, find out the frequency of each interarrival time in the trace in 0.001 second bins. Then plot these frequencies: put "Interarrival time in seconds" on the x-axis and "Number of times a value in this bin was observed" on the y-axis. Use a log10 scale for the y-axis, and a range of 0 to 0.1 seconds on the x-axis. Finally, on top of this plot, draw a line showing the expected frequency of each bin, if interarrival times are exponentially distributed with mean 0.005 seconds. 

**Lab report**: Creat the plot, and comment on your observations.

Finally, validate the assumption that the queue size is effectively infinite - make sure no packets are dropped. Look at the D-ITG receiver and sender logs, and make sure the same number of packets are received as are sent. Also look at the queue monitor output - make sure that the number of packets dropped is zero, or at least that it is the same at the beginning and end of the queue monitor output file (indicating that if any packets were dropped, they were dropped before the experiment began.

There is one more thing we need to verify: that, if we run the experiment repeatedly, we will have a valid stochastic experiment. Run the D-ITG experiment again with exactly the same arguments as before. Then, compare the sequence of packet sizes and interarrival times observed by tshark - are they the same from one experiment to the next, or are they different? 

**Lab report**: Comment on your observations.



### Putting it together

Putting together everything we have learned, we're going to measure the queue length of an M/M/1 queue for a range of ρ values in a testbed experiment. For convenience, we will keep everything constant and only vary λ from one experiment to the next.


You will run `ITGSend` with a mean packet size of 512 B (but use 470 B as the argument to D-ITG, to account for packet header overhead), and varying packet rate. Run experiments for the following values of λ: 225, 200, 175, 150, 125. For each value of λ you should run 5 experiments.

Given that you have already set up the queue as described above, to run a single experiment, you will:

* Record the requested λ and μ, and find ρ. 
* Start `ITGRecv` on the server node.
* Start `ITGSend` with the appropriate arguments and with a long experiment duration (e.g. 240 seconds) on the client node.
* Start the queue monitor script on the router node, and let it run until just before the experiment is over. (Don't let it keep running after the experiment is over.)
* Record the mean queue occupancy from your experiment.

**Lab report**: Complete this table for these values of λ: 225.0, 200.0, 175.0, 150.0, 125.0. In the "Mean queue occupancy - testbed" column, fill in the mean result across all repetitions from 1-5 for that value of ρ, *and* in parentheses, the standard deviation of the mean across all repititions from 1-5 for that value of ρ.

| ρ  | Mean queue occupancy - testbed | Mean queue occupancy - analytical model |
| ------------- | ------------- | ------------- |
| .. | .. | .. | ..
| .. | .. | .. | ..



**Lab report**: Create a plot with ρ on the horizontal axis and mean queue occupancy on the vertical axis. 

* Draw a red line representing the behavior predicted by the analytical model.
* Add the results of your testbed experiment at a series of blue markers.
* Add the results of your simulation experiment as a series of green markers.
