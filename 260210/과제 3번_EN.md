# 1. Questions
## 1) Why does the ALB exist in public sunbets
- ALB must handle traffic from the Internet
- Requests from user browsers come into the ALB through the Internet via the IGW and then properly distributed to web servers in private subnets

## 2) Why do Application Servers exist in Private subnets?
- If servers are accessible externally, unnecessary attack surfaces are exposed
- Servers should accept traffic only through the ALB, and their existence and related information must remain hidden from external sources

## 3) Why is a NAT Gateway necessary?
- The NAT Gateway controls outbound traffic from private subnets to external sources
- Limited Internet communication is required for server package updates

## 4) Why shouldn't RDS be placed in pubnet subnets?
- Since RDS is a managed database service, placing it in a pubblic subnet exposes the databse to the Internet
- Critical data such as customer information and administrator credentials stored in the dabase could be leaked

## 5) In which direction is the security group open?
- Security groups only allow explicit allow rules; anything not specified is denied by default
- All outbound traffic is allowed by default, while inbound traffic has no rules and is denied by default
- Inbound access control must be configured for allowed IPs and protocols
