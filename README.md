# jira-cachecar
A Sidecar/NGinx Caching solution for Jira Data Center to cache static objects/offload tomcat and java services.

# Customizing this repo
* Configs can easily be used and applied to Nginx running infront of Tomcat/Jira locally. This solutuon is desinged to be a local/node specific proxy behind a load balancer. LB > Nginx > Jira.
* Customize the nginx-test.conf or use this in your reference design/testing 
* Build your own docker image and use with Docker/Kubernetes and proxy traffic to your local jira node

# How does this work? 
Nginx basically plays with HTTP headers and caches incoming requests based on Jira's paths. Traffic from your load balancer is terminated at Nginx and then upstream/proxied to Tomcat/Jira on the localhost. 

Jira may need some configration adjustment to make this work.

# What's cached? What's the deal?
* /s/ contains a bunch of static resources, they're fairly persistent and are long lived including Javascript and CSS
* /secure/attachment is your jira's attachements, Nginx caches these if there is a HIT or HTTP 200 for 60s, reducing the requirement to retrieve attachements from NFS/Shared home on a per node basis. This should be adjusted and reduced based on use-case of attachements in Jira. In this scenario, a user may view an issue several times and the attachement is cached in the users browser/nginx cache for a duration.
* /images/icons/emoticons is just a bunch of other assets like .gifs and images servived like buttons and UI things, these can be cached for a longer duration


# Caveats? 
* Plugins/Changes to CSS may not be reflected quickly as they're cached. Commercial solutions like Nginx+ have cache expiry features that can help here. 

# Recommendations
Caches should be short, long lived caches could create a inconsistent experience or lead to end user frustration using older objects. Less than 8 hours would be recommended. Once Jira the user logs in and receives most of the assets for the day, this should offload jira enough to make this solution valuable for a large deployment. 

Caches can be blown away as per lifecycle of pods/AMIs or deployments.

Caches should live on local disks and have access to fast IO. Tuning Nginx is out of scope of this repo. Working with your nginx specialists and network teams on how to optimize and leverage Nginx is beyond the scope of this repo. 


