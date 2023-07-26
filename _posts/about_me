---
layout: post
title:  "About me"
categories: personal info
---
Some personal information
And also some code snippets:

{% highlight ruby %}
Using gcloud
# Allow traffic from IAP range to VM (ssh and/or RDP)
gcloud compute firewall-rules create allow-rdp-ingress-from-iap 
--allow=tcp:22,tcp:3389   --source-ranges=35.235.240.0/20


# Allow principal to use IAP at project level
gcloud projects add-iam-policy-binding PROJECT_ID 
--member=user:EMAIL  --role=roles/iap.tunnelResourceAccessor

gcloud projects add-iam-policy-binding PROJECT_ID 
--member=user:EMAIL --role=roles/compute.instanceAdmin.v1

# Or allow principal to use IAP at individual VM level 
gcloud compute instances add-iam-policy-binding VM
--member=user:EMAIL  --role=roles/iap.tunnelResourceAccessor

gcloud compute instances add-iam-policy-binding VM
--member=user:EMAIL --role=roles/compute.instanceAdmin.v1

# Connect to instance by ssh
ssh-add ~/.ssh/google_compute_engine
gcloud compute ssh VM  --zone ZONE --tunnel-through-iap

# Connect to instance by RDP
# Start a tunnel to RDP endpoint (port 3389 on VM)  to any local port
gcloud compute start-iap-tunnel VM 3389 --zone=ZONE --local-host-port=localhost:PORT 

# The command performs a connectivity test, and opens a tunnel to the selected port
# Output: Listening on port [PORT]

# Open the Microsoft Windows Remote Desktop Connection app.
# Enter the tunnel endpoint when asked for the computer name
localhost:PORT
# Use a valid VM’s username and password to connect
{% endhighlight %}

