# Bash Interface to Memcached

For many small projects an SQL database is overkill for simple storage of values. As an alternate solution there are a number of excellent distributed in-memory caching systems that can be used.

Memcached is an simple but fast in-memory caching system that has only a dozen commands so users can get up and running in no time.

There are documented API's in most of the common programming languages, with the exception of Bash. This project addresses how to use Bash to communicate with memcached.

## Getting Started with Memcached

Memached can be installed on all major OS's. To install it on Ubuntu/Raspberry Pi:
```shell
sudo apt-get install memcached
```
If you're using Docker there are some [lightweight memcached images (~89MB)](https://hub.docker.com/_/memcached) that can be used. 

To run the memcached docker image locally the **_--net host option_** should be used:
```bash
$ sudo docker run -it --net host memcached
```
The -it (interactive) option could be useful if there are any errors kicked out.

See the man pages for a full description of memcached options.

As a quick test Telnet can be used to enter manual commands (port 11211 is the default):
```bash
$ telnet localhost 11211
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
version
VERSION 1.6.12
 
lru_crawler metadump all
key=mynum exp=-1 la=1642363105 cas=2 fetch=no cls=1 size=66
END
quit
Connection closed by foreign host.
```

## Testing Bash with Memcached

The Bash **_nc_** command can be used to read and write to socket. The -q option will close the socket after 0 seconds (after the echo command is sent). The default port is 11211 but this can be changed along with adding some security options. 

A test example to read the memcached **_stats_** command locally would be:

```bash
$ echo "stats" | nc -q 0  172.0.0.1 11211 
STAT pid 1
STAT uptime 722
STAT time 1642359428
STAT version 1.6.12
...
END
```
## Set a Key and Value

To set an in-memory key-value store, the syntax is:
```
set mykey <flags> <ttl> <size>
value
```
The flags option is typical set to 0. The ttl “time to live” is in seconds, a ttl of 0 is indefinite.The size of the value also needs to be define. (This is taken care of in the Python, PHP… libraries). It’s important to note that a newline (\r\n) is required before the value.

To set a key to a variable mynum with a value of 55 and a indefinite time to live:
```bash
$ # Set a new key/value pair, with no flags and indefinite timeout
$ ipp="192.168.0.120 11211"
$ thekey="mykey1"
$ thenum=55
$ numsize=${#thenum}
$ # send command 
$ echo -e "set $thekey 0 0 $numsize\r\n$thenum\r" | nc -q 0  $ipp
STORED
```
## Get a Key and Value

The get key command returns 3 lines with the value being the 2nd line. Some awk code can be used to parse out just the value.
```bash
$ # Get a new key/value pair
$ ipp="192.168.0.120 11211"
$ thekey="mykey1"
$ 
$ echo "get $thekey" | nc -q 0  $ipp
VALUE mykey1 0 2
60
END
$ # Get just the 2nd line with the value
$ echo "get $thekey" | nc -q 0  $ipp | awk '{if (NR == 2) print $0}'
55
$ 
$ # Store the result in a variable
$ mykey1=$(echo "get $thekey" | nc -q 0  $ipp | awk '{if (NR == 2) print $0}')
$ echo "mykey1 = $mykey1"
mykey1 = 55
```
## Increment/Decrement a Value

The inc / decr commands will increase or decrease a numeric stored key value by a defined amount:
```bash
$ #incr/decr a value's number
#
ipp="192.168.0.120 11211"
thekey="mykey1"
 
# Get starting value
echo "get mynum" | nc -q 0 $ipp | awk '{if (NR == 2) print $0}'
55
 
thediff=10; # increase the value (only positive values)
# incr and show the value
echo "incr $thekey $thediff" | nc -q 0  $ipp
65
 
thediff=5; # decrease the value (only positive values)
# incr and show the value
echo "decr $thekey $thediff" | nc -q 0 $ipp
60
```

## Prepending and Appending

Memcached does not have queue or list functionality, if you need this take a look at Redis.

Below is an example script that creates a diagnostic log in a key/value. The append command adds msg text to the end of the overall string.
```bash
#!/usr/bin/bash
#
# diagmsg.sh - append msgs to a memcached variable string
#
ipp="192.168.0.120 11211"
msg="1:00 - Base Software Loaded\r"
size=${#msg}
 
echo -e "set mymsg 0 0 $size\r\n$msg\n\r" | nc -q 0  $ipp
 
# Create an array of diagnostics
diagmsgs=("2:00 - System Started\r" "2:15 - Getting Data\r" "2:30 - Backing up\r") 
# Append array to value text
for msg in "${diagmsgs[@]}"
do
  size=${#msg}
  #echo "Size: $size $msg";
  echo -e "append mymsg 0 0 $size\r\n$msg\n\r" | nc -q 0  $ipp;
done
 
# Show the result
echo -e "\nDiagnostic Results\n"
echo "get mymsg" | nc -q 0 $ipp 
```
The results of this script would be:
```bash
$ bash diagmsg.sh
STORED
STORED
STORED
STORED
 
Diagnostic Results
 
VALUE mymsg 0 92
1:00 - Base Software Loaded
2:00 - System Started
2:15 - Getting Data
2:30 - Backing up
 
END
```

## Summary

By adding a Bash interface to memcached it allows me to use programs like Octave/Matlib where a native interface isn’t available (just use the System call and pass the Bash code).


