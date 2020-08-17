


## Symptoms:
http requests to applications on EKS in the new VPC takes 10+sec from another account's VPC (not peered).

## Troubleshooting:
- The other account is connected to my account via TGW. Check if there was a problem there. Nothing conclusive on TGW to tell us that it's a packet routing issue. However, we were suspecting TGW since that was a new system outside our control. 
- Interestingly, when making request to the same services in the new VPC from another VPC which was peered, things were just fine; < 300msec response times.
- We tried doing few requests from our corporate network to see if there was a difference. Same behaviour as from another account; 10+ sec latency.
- At this point, we are pushing IT/Network and they don't have a clue either. We start collaborating.
- nslookup hostname returns in sub 100msec. So, we did not think that was an issue.
- However, we tried using the IP instead of the hostname in the URL. Surprise. <500msec response time. Given that DNS was not slow, we now started to consider if Host header was causing this issue. Hostname part of the URL gets set as Host header from HTTP 1.1. Now, We were suspecting if there was some host header validation that was causing the delay.
- Tried setting Host header with the IP and used the hostname based url. <500msec. Clearly, the host header is playing havoc.
- This set up has a LB Service Type on EKS. So, we wanted to understand what was happening on the LB side. LB Access logs clearly ruled out any kind of problem on the LB side. So, LB was fine. May be, something to do with the backend.
- My novice K8S skills were being pushed. I spent a couple of days understanding kubernetes networking and how it works. I got the basic hang of how the dance happens. Now back to problem.  
- After this, I started inspecting every little thing in my logs and the entire setup. I realized that the moment the request is made, it reaches instantenously to the pod in question (from logs) and the service also logs saying it got done with the request. It still was taking the same 10sec.
- Now, I had to drop one more level down. Let's dump packets - tcpdump! I started to look at how to setup tcpdump for pods and I found a neat way to run tcpdump as another container within the same pod (I also learnt about kubectl patch!)
- After patching up the deployment, I got the tcpdump to run; but, it took a couple of iterations before I settled on what I wanted.
- I logged syn/fin packets + http request packets alone and I made a request.
- Now, I see a connection from the host which ran my curl and the corresponding HTTP headers. Good. One step forward.
- First, I see a msg with tcpflag P (Push) + request data (headers and stuff)
- And then, ~5.5 seconds later, I see the tcpflag F (Fin) from my server sent to the client
- Followed by an ACK for the Fin msg from the client. The sequence numbers all line up.
- And, with this setup, I also validated that my server's accesslog indicate the request was serviced a few millis after the packet with tcpflag (P) arrived.
- I then extended my packet capture rules to log all packets; since I wanted to understand if & how the hostname was affecting the behaviour. 
- now, I see a few more things... The server receives a SYN, acks it, and then it gets the request payload, to which it responds with the actual response with response payload (all within a few msec), the client responds back to the response with an ACK... At this point, we expect the server to send a FIN to close the connection. interestingly, the server waits for 5 secs before it sends a FIN. This is very strange. Wondering, if this is due to keep-alive or other settings...
- All this packet capture logs are not helping us move forward. Clearly, there's someone sitting in between and processing the packets. The network team believes that there's no L7 system in between who pokes into these Host headers. Left with no choice at this point, but, to reach out to AWS and see what they have to say
- With AWS in the picture, I'm trying to pull together 4 different teams towards a solution. My AWS Support ticket had all the information with notes that packet capture was done. I did not share everything upfront, because, I wasn't sure if it would be overwhelming. In one/two roundtrips, we get to a conclusion that there's nothing happening on the AWS side - in terms of VPC networking or anything else. 
- By now, we had also eliminated all the intermediaries - the ELB, K8S and stuff by setting up a standalone EC2 instance and having an A-record to it. Since, the behaviour was reproducible, it was clear that this was a infra(network/vpc/L7) issue.
- I had also been reaching out to the network team with the AWS support conversation notes and had them thinking about what was going on. Finally, someone from our internal networking team left me a note - "Hey, I may have found the issue. Wait"
- Turns out, the firewall (which was supposed to be max L4) had an optional feature, which was enabled, which did host based checks. This component contacts another service - which provides a verdict on the host maliciousness. That service was not responding and the firewall client was not handling the failure properly. It would allow traffic one way and hold back responses until the client retries were exhausted and then "allow" the traffic. Duh! How many times... How many times... Errorrrr handling!!!
- The network team made the service active, tested a few things and then set it back up. Latencies are back to a few 100msec.

## Lessons Learnt
- Bringing teams together to arrive at a solution is more important. When multiple teams are involved, there's a need for someone who's agenda is the solution, instead of taking sides
- Isolating the problem is super critical - It was important to remove the moving parts and bring it down to bare minimum by setting up a dummy instance serving http traffic without any intermediary layers.
- When in doubt, do not assume. Always clarify - Even after so many rounds of probing, we still assumed the firewall did not have any L5 and above features and that proved to be a costly assumption.
- Proactive measures - when migrating systems to new networks, _always_ _always_ test out connectivity, latency, DNS resolution from target (consumer) environments first before migrating apps
- Keep Calm & Debug!
