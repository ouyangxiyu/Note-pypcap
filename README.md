
# Background
* Some notes when study on https://github.com/dugsong/pypcap.git    
* Me, once being a code typist, only familiar with C, A newbie for Python.
* All the code run on the mac book pro, Sierra, Python 2.7. 'en4' is my usb ethernet device.


# Section 1
The API from pcap.pyx, focus on the code

## findalldevs
* Description  
  http://www.tcpdump.org/manpages/pcap_findalldevs.3pcap.html

  in pypcap just return the name[], not as in libpcap, return the struct pcap_if_t

* Code
```
    print ("The interfaces in the system are:")
    devs = pcap.findalldevs()
    print devs
```
* Output
```
    ['bridge0', 'utun0', 'en1', 'utun1', 'en2', 'en4', 'lo0', 'en0', 'gif0', 'stf0', 'p2p0', 'awdl0']
```
* Notes

  maybe used to find and choose which interface to capture the packets  

## lookupdev   
* Description  
  http://www.tcpdump.org/manpages/pcap_lookupdev.3pcap.html

  return the first valid interface to capture the packets

* Code
```
    print ("The first interface in the system is:")
    dev = pcap.lookupdev()
    print dev
```
* Output
```
    bridge0
```
* Notes

  if you call pcap.pcap(), and no packets are output, you should first check which
  interface you are capturing on.

## ex_name
* Description  
  nothing, libpcap has no such API

* Code
```
dev = pcap.ex_name('en4')
print dev
```
* Output
```
en4
```
* Notes
  from the code, pcap_ex.c, this api only for wincap, so just ignore it for now.


## lookupnet
  * Description  
    http://www.tcpdump.org/manpages/pcap_lookupnet.3pcap.html

    return the net_address and mask_address, but not the gateway_address
  * Code-1

    ```
     p = pcap.lookupnet('en4')
     print p
     ```

  * Output-1
    ```
    ('\xc0\xa8\xc1\x00', '\xff\xff\xff\x00')
    ```

  * Code-2
    ```
    netp,maskp = pcap.lookupnet('en4')
    a = struct.unpack('BBBB',netp)
    b = struct.unpack('BBBB', maskp)

    print "Net_addr: %d.%d.%d.%d" %(a[0],a[1],a[2],a[3])
    print "Msk_addr: %d.%d.%d.%d" %(b[0],b[1],b[2],b[3])
    ```
  * Output-2
    ```
    Net_addr: 192.168.193.0
    Msk_addr: 255.255.255.0
    ```
  * Notes
    Not as libpcap, return a packed struct, contains net_address and mask_address
    If wanna print it as a IP address format, I think the code-2 is easy to
    understand.

## pcap
* Description
   there are lots of demo codes, written as pcap.pcap() then balabala.
   but there is no such API in libpcap.
   in my opinion，it should be pcap_open_live()
   http://www.tcpdump.org/manpages/pcap_open_live.3pcap.html
* Code-1

  ```
  p = pcap.pcap('en4')
  print p
  ```
* Output-1
  ```
  <pcap.pcap object at 0x10391c4f0>
  ```
* Code-2

  ```
  p = pcap.pcap('en4')
  for timestamp, buf in p:
      print 'Timestamp: ', str(datetime.datetime.utcfromtimestamp(timestamp))
      #print some pkt
  ```
* Output-2
  ```
  <pcap.pcap object at 0x108c434f0>
  Timestamp:  2016-12-13 06:25:13.815806
  Timestamp:  2016-12-13 06:25:13.815982
  Timestamp:  2016-12-13 06:25:13.815984
  Timestamp:  2016-12-13 06:25:13.815985
  ^C
  Traceback (most recent call last):
    File "findalldevs.py", line 18, in <module>
      main()
    File "findalldevs.py", line 13, in main
      for timestamp, buf in p:
    File "pcap.pyx", line 384, in pcap.pcap.__next__ (pcap.c:4108)
      raise KeyboardInterrupt
  KeyboardInterrupt
  ```
* Code-3
  ```
  p = pcap.pcap('bridge0')
  print p.getnonblock()
  print p.__next__()

  for timestamp,buf in p:
      print 'Timestamp: ', str(datetime.datetime.utcfromtimestamp(timestamp))
      time.sleep(1)
  ```

* Output-3
```
Password:
False
^C^C^C^Z
[3]+  Stopped                 sudo python findalldevs.py
```

* Code-4
```
  p = pcap.pcap('bridge0')
  p.setnonblock()
  print p.getnonblock()

  print p.__next__()

  for timestamp,buf in p:
      print 'Timestamp: ', str(datetime.datetime.utcfromtimestamp(timestamp))
      time.sleep(1)
```
* Output-4
```
True
None
Traceback (most recent call last):
  File "findalldevs.py", line 21, in <module>
    main()
  File "findalldevs.py", line 16, in main
    for timestamp,buf in p:
TypeError: 'NoneType' object is not iterable
```

* Notes
   It costs me much time on figuring out why Code-2 can output the packet all
   the time. At last I found the answer in useage of For, a iterative object
   in For,as "pcap", should call the method "next" for iterative, so it will
   output the packet until "KeyboardInterrupt" or other exceptions.
   But there are some question still unresolved as：
   ** setnonblock, I think set the select parameters with nonblock, but the code in
      pcap_ex.c, struct timeval tv = { 1, 0 }, so it seems work as nonblock always.
      in pcap.c there are some codes for setnonblock, still do not understand how
      it works. [Please help]

   In the end, pcap.pcap()， it returns a handle, all the packets captured are buffered here.
   you can use any iterative methods to get all the packets. as "For" or "pcap_loop" or "pcap_dispath"


 ## pcap_dispath/pcap_loop
 * Description  
   http://www.tcpdump.org/manpages/pcap_loop.3pcap.html

 * Code

 ```
 n = 0
 
 def pkt_cb(tv, pkt, arg):
     print "Got a packet"
     print 'Timestamp: ', str(datetime.datetime.utcfromtimestamp(tv))
     global n
     n = n + 1


 def main():

     p = pcap.pcap('en0')


     try:
         pcap.pcap.dispatch(p, 0, pkt_cb, 0)
         #pcap.pcap.loop(p,0, pkt_cb, 0)
     except:
         print n
 ```



 * Output

 ```
 Got a packet
 Timestamp:  2016-12-14 01:53:58.815039
 Got a packet
 Timestamp:  2016-12-14 01:53:58.860533
 Got a packet
 Timestamp:  2016-12-14 01:53:58.860595
 Got a packet
 Timestamp:  2016-12-14 01:53:59.813851
 51
 ```

 * Notes
   the different between pcap_loop and pcap_dispatch，I think pcap_loop use
   method of next to loop till loopbreak or some exceptions,
   attention if no packets, it will not break the loop. But pcap_dispatch
   will break when no more packets arrived.
