---
layout: post
title:  "Upping the home monitoring game"
date:   2024-03-31 00:00:00 -0800
categories: infrastructure hobby
---
This article documents my work of building a reliable home internet distruptor detector starting from local monitoring and ending up with an AWS backed monitoring solution.

Recently I learned my home security system is in fact, not secure.
My home network infra wrt security systems is pretty standard:
![image](/assets/images/home_monitoring/FTHomeMonitoring.jpg)  
If someone targets my home and uses a jammer while I am away, I'd have no clue.
Unfortunately migrating to a wired solution would be costly, so taking a step back, at least I'd like to be aware if something is going on. 

A quick search yields uptime-kuma, and it's perfect for what I want to do.
Getting it up and running on the ubuntu server takes less than 15 minutes, where 10 minutes is installing docker getting its permissions right.
![image](/assets/images/home_monitoring/kuma.png)
[Uptime kuma](https://github.com/louislam/uptime-kuma)
1. install docker [https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
2. configure docker permissions (unless you want to sudo) [https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)
3. pull uptime-kuma image and run it `docker run -d --restart=always -p 3001:3001 -v uptime-kuma:/app/data --name uptime-kuma louislam/uptime-kuma:1`
4. configure monitors
5. create discord server(chatroom)
6. create discord webhook e.g. [https://www.svix.com/resources/guides/how-to-make-webhook-discord/](https://www.svix.com/resources/guides/how-to-make-webhook-discord/)
7. attach discord webhook to monitor

Monitoring configuration is straightforward. Here's mine:
```
Monitor Type > ping
Hostname > IP address of home device (for devices like ring, it almost never changes. My ring IP address was stable for the last 2 years. If your ip address tend to change, you can use mac-IP binding in router configuration)
Interval > 60
Retries > 1
Heartbeat Retry Interval > 20
Resend Notification if Down X times consecutively (Resend disabled) > 5
Notifications > Discord
```


On notification options, I chose discord webhooks because it supports mobile push notifications, so that I can get a pop-up on my phone the moment uptime-kuma alerts. It's also the only free option that supports push notifications, compared to the other options I explored (slack may support free push notifications too, but I use slack for work and it's muted on my personal phone -- maybe the takeaway is I shouldn't install work slack on my personal phone?). 

![image](/assets/images/home_monitoring/Kuma-Discord.jpg) 

![image](/assets/images/home_monitoring/kuma-alarms.PNG){: width="50%"}

------

This seemed satisfactory, until I realized that power to my house could be cut. The external cables of the house are a vulnerable attack vector -- I know someone whose residence was breached by lawbreakers who severed the electricity, rendering all security systems inoperative. I have a backup power supply for my wired network servers, so the fix is easy -- given the abundance of IoT devices everywhere, just have uptime-kuma ping one of these as a proxy for power status. I chose to ping my air filter and one of my alexa devices.

------

Now I would get notifications if my wifi is jammed, or my power is cut, cool .... until I remembered that my internet cable is on the OUTSIDE of my house, along with the power cables. One option is to have a cellular backup for redundant paths to the internet, but it's an expensive solution. My solution is to use a cloud provider: Have my home server send heartbeat metrics to the cloud, and alarm when metrics are missing. 

![image](/assets/images/home_monitoring/full.jpg) 

1. Setup a server user in AWS account. 
2. Give it IAM permissions (cloudwatch putmetrics + setalarmstate) . 
3. Install awscli in home server. [https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
4. Setup access key the user, and configure awscli

Send a heartbeat metric to cloudwatch and confirm its being picked up:
```
aws cloudwatch put-metric-data --namespace HomeServer --metric-name Heartbeat --unit Count --value 1
```
![image](/assets/images/home_monitoring/cw-metrics.png) 

Set the put metric cli command to trigger every minute:
1. linux shell: `crontab -e`
2. In crontab editor, add the command that was just tested
```
* * * * * aws cloudwatch put-metric-data --namespace HomeServer --metric-name Heartbeat --unit Count --value 1
```
3. linux shell: `grep CRON /var/log/syslog` to ensure cron is triggering correctly

Now having a periodic metric to cloudwatch, alarm on missing metrics.

![image](/assets/images/home_monitoring/cw-alarm.png) 

My setup was to alarm when there is less than 1 datapoint for 1 minute.
```
Metric name > Heartbeat
Statistic > Sum
Period > 1 min
Threshold Type > Static
Whenever threshold is > Lower
Than .. > 1
Additional Configuration:
Datapoints to alarm > 1 out of 1
Missing data treatment > Treat missing data as bad !!! This is critical because if the home 
                                   server's connection is cut, the metric data will be missing
                                   and that needs to trigger the alarm
Alarm Action > SNS topic * see below

```

On how to receive the alarm notification, I have the alarm send to an SNS topic, and have a lambda and email subscription to the topic. The lambda can publish notifications to the same discord webhook I used for uptime-kuma.
1. Create SNS Topic
2. Create Lambda that subscribes to SNS topic
3. Configure the alarm to send to SNS topic

There's tons of material on this classic integration pattern, here's one for consideration [https://docs.aws.amazon.com/lambda/latest/dg/with-sns-example.html](https://docs.aws.amazon.com/lambda/latest/dg/with-sns-example.html)

Here's my lambda. I didn't even bother parsing the message because receiving an SNS message = alarm went off. Maybe I'll polish it in future iterations...
```python
import json
from urllib.request import Request, urlopen

webhook_url = 'https://discord.com/api/webhooks/...'
title_format = ':x: {} :x:'
message_format = '''
* Possible internet or power outage
Check ring / alexa
'''

def lambda_handler(event, context):
    #print("Received event: " + json.dumps(event, indent=2))
    message = event['Records'][0]['Sns']['Message']
    print("From SNS: " + message)
    data = {
        'username' : 'Cloudwatch Alarms',
        'embeds': [{
            'title': title_format.format('Home Server Heartbeat Alarm'),
            'color': 15548997,
            'description': message_format
        }],
    }
    # Convert your data to JSON
    json_data = json.dumps(data).encode('utf-8')
    # Create a request object
    req = Request(webhook_url, data=json_data, headers={'Content-Type': 'application/json', 'User-Agent': 'AWS'})
    response = urlopen(req)
    # Read response data
    response_data = response.read()
    print(response_data)
    return message
```

Finally, test the alarm using this command on the server:
```
aws cloudwatch set-alarm-state --alarm-name "Missing Home Server Heartbeat" --state-value ALARM --state-reason "testing purposes"
```
![image](/assets/images/home_monitoring/cw-alarms.PNG){: width="50%"}


There. I think my monitoring system is finally resilient to the known attacks. There are still holes in this monitoring solution (or more accurately, a monitor of a home monitoring system), such as a single home server is not fault-tolerant, and the solution is suceptible to regional aws outages, but it's a reliable enough for the purpose of an early warning that something is amiss. 

To summarize, I started with a LAN-based solution, and even with UPS, the physical network link outside my house is still a weak link. To mitigate that, I added a low-cost AWS based solution that adds a great deal of reliability.


-------
[https://www.wxyz.com/news/how-criminals-are-using-jammers-deauthers-to-disrupt-wifi-security-cameras](https://www.wxyz.com/news/how-criminals-are-using-jammers-deauthers-to-disrupt-wifi-security-cameras)  
[https://www.npr.org/../washington-electricity-power-grid-sabotage-attacks-blackout-outage](https://www.npr.org/2023/01/04/1146889176/washington-electricity-power-grid-sabotage-attacks-blackout-outage#:~:text=Federal%20agents%20say%20one%20of,both%20residents%20of%20Puyallup%2C%20Wash.)  
[https://leovoel.github.io/embed-visualizer/](https://leovoel.github.io/embed-visualizer/)  
[https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-data.html](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-data.html)  
[https://birdie0.github.io/discord-webhooks-guide/structure/embeds.html](https://birdie0.github.io/discord-webhooks-guide/structure/embeds.html)