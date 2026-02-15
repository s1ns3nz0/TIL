# 1. Questions
## 1) What is the first component user encounters when entering a URL?
- AWS Route53(DNS service)
    + The URL entered in the brower must be converted to an IP address to send a request to the server
    + Quering the DNS server is required to resolve the URL to an IP address
    + During the recursive DNS query process, The authoritative name server for the requested website is queried
## 2) Where is HTTPS terminated?
- HTTPS is typically terminated at the load balancer in front of the web server
- Communication between the load balancer and the server is considered a truted zone. While Encryption can be applied to this segment, it may burden the server
## 3) Where are static resources and API requests seperated?
- When using a CDN, static resources are served from a S3 bucket, while resources to /api is sent to the ALB
- The ALB uses path-based routing to direct specific paths to servers hosting static files and others to API servers
## 4) What criteria determine server scaling?
- Monitor core metrics through CloudWatch
    + CPU Usage: The most common metric; scales when usage exceeds a specific threshold
    + The number of requests: Scales when requests per unit time exceeds a threshold
    + Memory USage: Scales when memory usage is high, depending on the application's memory characteristcs
## 5) At what point are failures detected?
- ELB Health CHeck: Load balancer periodically sends health check signals and excludes unresponsive servers from the traffic pool
- CloudWatch Alarms: Detects issues such as resource exhaustion and errors through alarms triggered by monitored metrics
- Route 53 Health Check: Periodically sends requests to specific IP addresses and domains to verify their availability
    + HTTP, HTTPS: Accesses specific URLs and waits for responses
    + TCP: Verifies that specific ports on the EC2 instace are open