# coLunacyDNS

coLunacyDNS is a simply IPv4-only forwarding DNS server controlled
by a Lua script.  It allows a lot of flexibility because it uses
a combination of C for high performance and Lua for maximum
flexibility.

# Getting started

On a CentOS 8 Linux system, this gets us started:

```bash
./compile.coLunacyDNS.sh
su
./coLunacyDNS -d
```

Here, we use `coLunacyDNS.lua` as the configuration file.

Since coLunacyDNS runs on port 53, we need to start it as root.
As soon as coLunacyDNS binds to port 53 and seeds its internal 
secure pseudo random number generator, it calls `chroot` and drops
root privileges.

Cygwin users may use `compile.cygwin.sh` to compile coLunacyDNS, since
Cygwin does not have the same sandboxing Linux has.

# Configration file examples

In this example, we listen on 127.0.0.1, and, for any IPv4 query,
we return the IP of that query as reported by 9.9.9.9.

```lua
bindIp = "127.0.0.1" -- We bind the server to the IP 127.0.0.1
function processQuery(Q) -- Called for every DNS query received
   -- Connect to 9.9.9.9 for the query given to this routine
   t = coDNS.solve({name=Q.coQuery, type="A", upstreamIp4="9.9.9.9"})
   -- Return a "server fail" if we did not get an answer
   if(t.error or t.status ~= 1) then return {co1Type = "serverFail"} end
   -- Otherwise, return the answer
   return {co1Type = "A", co1Data = t.answer}
end
```

As an even simpler example, we always return "10.1.1.1" for any DNS
query given to us:

```lua
bindIp = "127.0.0.1" -- We bind the server to the IP 127.0.0.1
function processQuery(Q) -- Called for every DNS query received
  return {co1Type = "A", co1Data = "10.1.1.1"}
end
```

Here is a more complicated example:

```lua
-- coLunacyDNS configuration
bindIp = "127.0.0.1" -- We bind the server to the IP 127.0.0.1

-- Examples of three API calls we have: timestamp, rand32, and rand16
coDNS.log(string.format("Timestamp: %.1f",coDNS.timestamp())) -- timestamp
coDNS.log(string.format("Random32: %08x",coDNS.rand32())) -- random 32-bit num
coDNS.log(string.format("Random16: %04x",coDNS.rand16())) -- random 16-bit num
-- Note that it is *not* possible to use coDNS.solve here; if we attempt
-- to do so, we will get an error with the message
-- "attempt to yield across metamethod/C-call boundary".  

function processQuery(Q) -- Called for every DNS query received
  -- Because this code uses multiple co-routines, always use "local"
  -- variables
  local returnIP = nil
  local upstream = "9.9.9.9"

  -- Log query
  coDNS.log("Got IPv4 query for " .. Q.coQuery .. " from " ..
            Q.coFromIP .. " type " ..  Q.coFromIPtype) 

  -- We will use 8.8.8.8 as the upstream server if the query ends in ".tj"
  if string.match(Q.coQuery,'%.tj%.$') then
    upstream = "8.8.8.8"
  end

  -- We will use 4.2.2.1 as the upstream server if the query comes from 
  -- 192.168.99.X
  if string.match(Q.coFromIP,'^192%.168%.99%.') then
    upstream = "4.2.2.1"
  end

  -- Right now, coLunacyDNS can *only* process "A" (IPv4 IP) queries
  if Q.coQtype ~= 1 then -- If it is not an A (ipv4) query
    -- return {co1Type = "ignoreMe"} -- Ignore the query
    return {co1Type = "notThere"} -- Send "not there" (like NXDOMAIN)
  end

  -- Contact another DNS server to get our answer
  t = coDNS.solve({name=Q.coQuery, type="A", upstreamIp4=upstream})

  -- If coDNS.solve returns an error, the entire processQuery routine is
  -- "on probation" and unable to run coDNS.solve() again (if an attempt
  -- is made, the thread will be aborted and no DNS response sent 
  -- downstream).  
  if t.error then	
    coDNS.log(t.error)
    return {co1Type = "serverFail"} 
  end

  -- Status is 1 when we get an IP from the upstream DNS server, otherwise 0
  if t.status == 1 and t.answer then
    returnIP = t.answer
  end

  if string.match(Q.coQuery,'%.invalid%.$') then
    return {co1Type = "A", co1Data = "10.1.1.1"} -- Answer for anything.invalid
  end
  if returnIP then
    return {co1Type = "A", co1Data = returnIP} 
  end
  return {co1Type = "serverFail"} 
end
```

# Security considerations

Since the Lua file is executed as root, some effort is made to restrict
what it can do:

* Only the `math`, `string`, and `bit32` libraries are loaded from
  Lua's standard libs.  (bit32 actually is another Bit library, but with a
  `bit32` interface.)
* A special `coDNS` library is also loaded.
* The Lua script should not have access to the filesystem nor be able
  to do anything malicious.
