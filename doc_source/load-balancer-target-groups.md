# Target Groups for Your Network Load Balancers<a name="load-balancer-target-groups"></a>

Each *target group* is used to route requests to one or more registered targets\. When you create a listener, you specify a target group for its default action\. Traffic is forwarded to the target group specified in the listener rule\. You can create different target groups for different types of requests\. For example, create one target group for general requests and other target groups for requests to the microservices for your application\. For more information, see [Network Load Balancer Components](introduction.md#network-load-balancer-components)\.

You define health check settings for your load balancer on a per target group basis\. Each target group uses the default health check settings, unless you override them when you create the target group or modify them later on\. After you specify a target group in a rule for a listener, the load balancer continually monitors the health of all targets registered with the target group that are in an Availability Zone enabled for the load balancer\. The load balancer routes requests to the registered targets that are healthy\. For more information, see [Health Checks for Your Target Groups](target-group-health-checks.md)\.

**Topics**
+ [Routing Configuration](#target-group-routing-configuration)
+ [Target Type](#target-type)
+ [Registered Targets](#registered-targets)
+ [Target Group Attributes](#target-group-attributes)
+ [Deregistration Delay](#deregistration-delay)
+ [Proxy Protocol](#proxy-protocol)
+ [Create a Target Group for Your Network Load Balancer](create-target-group.md)
+ [Health Checks for Your Target Groups](target-group-health-checks.md)
+ [Register Targets with Your Target Group](target-group-register-targets.md)
+ [Tags for Your Target Group](target-group-tags.md)
+ [Delete a Target Group](delete-target-group.md)

## Routing Configuration<a name="target-group-routing-configuration"></a>

By default, a load balancer routes requests to its targets using the protocol and port number that you specified when you created the target group\. Alternatively, you can override the port used for routing traffic to a target when you register it with the target group\.

Target groups for Network Load Balancers support the following protocols and ports:
+ **Protocols**: TCP, TLS, UDP, TCP\_UDP
+ **Ports**: 1\-65535

The following table summarizes the supported combinations of listener protocol and target group settings\.


| Listener Protocol | Target Group Protocol | Target Group Type | Health Check Protocol | 
| --- | --- | --- | --- | 
| TCP | TCP \| TCP\_UDP | instance \| ip | HTTP \| HTTPS \| TCP | 
| TLS | TCP \| TLS | instance \| ip | HTTP \| HTTPS \| TCP | 
| UDP | UDP \| TCP\_UDP | instance | HTTP \| HTTPS \| TCP | 
| TCP\_UDP | TCP\_UDP | instance | HTTP \| HTTPS \| TCP | 

## Target Type<a name="target-type"></a>

When you create a target group, you specify its target type, which determines how you specify its targets\. After you create a target group, you cannot change its target type\.

The following are the possible target types:

`instance`  
The targets are specified by instance ID\.

`ip`  
The targets are specified by IP address\.

When the target type is `ip`, you can specify IP addresses from one of the following CIDR blocks:
+ The subnets of the VPC for the target group
+ 10\.0\.0\.0/8 \(RFC 1918\)
+ 100\.64\.0\.0/10 \(RFC 6598\)
+ 172\.16\.0\.0/12 \(RFC 1918\)
+ 192\.168\.0\.0/16 \(RFC 1918\)

**Important**  
You can't specify publicly routable IP addresses\.

These supported CIDR blocks enable you to register the following with a target group: ClassicLink instances, AWS resources that are addressable by IP address and port \(for example, databases\), and on\-premises resources linked to AWS through AWS Direct Connect or a software VPN connection\.

When the target type is `ip`, the load balancer can support 55,000 simultaneous connections or about 55,000 connections per minute to each unique target \(IP address and port\)\. If you exceed these connections, there is an increased chance of port allocation errors\. If you get port allocation errors, add more targets to the target group\.

If the target group protocol is UDP or TCP\_UDP, the target type must be `instance`\.

Network Load Balancers do not support the `lambda` target type, only Application Load Balancers support the `lambda` target type\. For more information, see [Lambda Functions as Targets](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/lambda-functions.html) in the *User Guide for Application Load Balancers*\.

### Request Routing and IP Addresses<a name="request-routing-ip-addresses"></a>

If you specify targets using an instance ID, traffic is routed to instances using the primary private IP address specified in the primary network interface for the instance\. The load balancer rewrites the destination IP address from the data packet before forwarding it to the target instance\.

If you specify targets using IP addresses, you can route traffic to an instance using any private IP address from one or more network interfaces\. This enables multiple applications on an instance to use the same port\. Note that each network interface can have its own security group\. The load balancer rewrites the destination IP address before forwarding it to the target\.

For more information allowing traffic to your instances, see [Target Security Groups](target-group-register-targets.md#target-security-groups)\.

### Source IP Preservation<a name="source-ip-preservation"></a>

If you specify targets using an instance ID, the source IP addresses of the clients are preserved and provided to your applications\.

If you specify targets by IP address, the source IP addresses are the private IP addresses of the load balancer nodes\. If you need the IP addresses of the clients, enable Proxy Protocol and get the client IP addresses from the Proxy Protocol header\.

If you have micro services on instances registered with a Network Load Balancer, you cannot use the load balancer to provide communication between them unless the load balancer is internet\-facing or the instances are registered by IP address\. For more information, see [Connections time out for requests from a target to its load balancer](load-balancer-troubleshooting.md#loopback-timeout)\.

## Registered Targets<a name="registered-targets"></a>

Your load balancer serves as a single point of contact for clients and distributes incoming traffic across its healthy registered targets\. Each target group must have at least one registered target in each Availability Zone that is enabled for the load balancer\. You can register each target with one or more target groups\. You can register each EC2 instance or IP address with the same target group multiple times using different ports, which enables the load balancer to route requests to microservices\.

If demand on your application increases, you can register additional targets with one or more target groups in order to handle the demand\. The load balancer starts routing traffic to a newly registered target as soon as the registration process completes\.

If demand on your application decreases, or you need to service your targets, you can deregister targets from your target groups\. Deregistering a target removes it from your target group, but does not affect the target otherwise\. The load balancer stops routing traffic to a target as soon as it is deregistered\. The target enters the `draining` state until in\-flight requests have completed\. You can register the target with the target group again when you are ready for it to resume receiving traffic\.

If you are registering targets by instance ID, you can use your load balancer with an Auto Scaling group\. After you attach a target group to an Auto Scaling group, Auto Scaling registers your targets with the target group for you when it launches them\. For more information, see [Attaching a Load Balancer to Your Auto Scaling Group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/attach-load-balancer-asg.html) in the *Amazon EC2 Auto Scaling User Guide*\.

**Limits**
+ You cannot register instances by instance ID if they have the following instance types: C1, CC1, CC2, CG1, CG2, CR1, G1, G2, HI1, HS1, M1, M2, M3, and T1\. You can register instances of these types by IP address\.
+ You cannot register instances by instance ID if they are in a VPC that is peered to the load balancer VPC\. You can register these instances by IP address\.

## Target Group Attributes<a name="target-group-attributes"></a>

The following are the target group attributes:

`deregistration_delay.timeout_seconds`  
The amount of time for Elastic Load Balancing to wait before changing the state of a deregistering target from `draining` to `unused`\. The range is 0\-3600 seconds\. The default value is 300 seconds\.

`proxy_protocol_v2.enabled`  
Indicates whether Proxy Protocol version 2 is enabled\. By default, Proxy Protocol is disabled\.

## Deregistration Delay<a name="deregistration-delay"></a>

When you deregister an instance, the load balancer stops creating new connections to the instance\. The load balancer uses connection draining to ensure that in\-flight traffic completes on the existing connections\. If the deregistered instance stays healthy and an existing connection is not idle, the load balancer can continue to send traffic to the instance\. To ensure that existing connections are closed, you can ensure that the instance is unhealthy before you deregister it, or you can periodically close client connections\.

The initial state of a deregistering target is `draining`\. By default, the load balancer changes the state of a deregistering target to `unused` after 300 seconds\. To change the amount of time that the load balancer waits before changing the state of a deregistering target to `unused`, update the deregistration delay value\. We recommend that you specify a value of at least 120 seconds to ensure that requests are completed\.

**To update the deregistration delay value using the console**

1. Open the Amazon EC2 console at [https://console\.aws\.amazon\.com/ec2/](https://console.aws.amazon.com/ec2/)\.

1. On the navigation pane, under **LOAD BALANCING**, choose **Target Groups**\.

1. Select the target group\.

1. Choose **Description**, **Edit attributes**\.

1. Change the value of **Deregistration delay** as needed, and then choose **Save**\.

**To update the deregistration delay value using the AWS CLI**  
Use the [modify\-target\-group\-attributes](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify-target-group-attributes.html) command\.

## Proxy Protocol<a name="proxy-protocol"></a>

Network Load Balancers use Proxy Protocol version 2 to send additional connection information such as the source and destination\. Proxy Protocol version 2 provides a binary encoding of the Proxy Protocol header\. The load balancer prepends a proxy protocol header to the TCP data\. It does not discard or overwrite any existing data, including any proxy protocol headers sent by the client or any other proxies, load balancers, or servers in the network path\. Therefore, it is possible to receive more than one proxy protocol header\. Also, if there is another network path to your targets outside of your Network Load Balancer, the first proxy protocol header might not be the one from your Network Load Balancer\.

If you specify targets by IP address, the source IP addresses provided to your applications are the private IP addresses of the load balancer nodes\. If your applications need the IP addresses of the clients, enable Proxy Protocol and get the client IP addresses from the Proxy Protocol header\.

If you specify targets by instance ID, the source IP addresses provided to your applications are the client IP addresses\. However, if you prefer, you can enable Proxy Protocol and get the client IP addresses from the Proxy Protocol header\.

### Health Check Connections<a name="health-check-connections"></a>

After you enable Proxy Protocol, the Proxy Protocol header is also included in health check connections from the load balancer\. However, with health check connections, the client connection information is not sent in the Proxy Protocol header\.

### VPC Endpoint Services<a name="custom-tlv"></a>

For traffic coming from service consumers through a [VPC endpoint service](https://docs.aws.amazon.com/vpc/latest/userguide/endpoint-service.html), the source IP addresses provided to your applications are the private IP addresses of the load balancer nodes\. If your applications need the IP addresses of the service consumers, enable Proxy Protocol and get them from the Proxy Protocol header\.

The Proxy Protocol header also includes the ID of the endpoint\. This information is encoded using a custom Type\-Length\-Value \(TLV\) vector as follows\.

[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html)

For an example that parses TLV type 0xEA, see [https://github\.com/aws/elastic\-load\-balancing\-tools/tree/master/proprot](https://github.com/aws/elastic-load-balancing-tools/tree/master/proprot)\.

### Enable Proxy Protocol<a name="enable-proxy-protocol"></a>

Before you enable Proxy Protocol on a target group, make sure that your applications expect and can parse the Proxy Protocol v2 header, otherwise, they might fail\. For more information, see [PROXY protocol versions 1 and 2](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)\.

**To enable Proxy Protocol using the console**

1. Open the Amazon EC2 console at [https://console\.aws\.amazon\.com/ec2/](https://console.aws.amazon.com/ec2/)\.

1. On the navigation pane, under **LOAD BALANCING**, choose **Target Groups**\.

1. Select the target group\.

1. Choose **Description**, **Edit attributes**\.

1. Select **Enable proxy protocol v2**, and then choose **Save**\.

**To enable Proxy Protocol using the AWS CLI**  
Use the [modify\-target\-group\-attributes](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify-target-group-attributes.html) command\.