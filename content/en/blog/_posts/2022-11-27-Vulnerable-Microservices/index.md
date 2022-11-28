---
layout: blog
title: "Vulnerable Microservices"
date: 2022-11-27
slug: vulnerable-microservices
---


**Author:** 
David Hadas (IBM Research Labs)

As cyber war games continue to intensify in sophistication, organizations deploying cloud services, continue to grow their cyber investments aiming to produce safe and non-vulnerable services. However, the year-by-year growth in cyber investments, does not result in parallel reduction in cyber incidents. Instead, the number of cyber incidents continue to grow annually. Evidently organizations are fighting a battle they are doomed to fail at. No matter how much effort is made to detect and remove cyber weaknesses from deployed services, it seems offenders have the upper hand.

Considering the current spread of offensive tools, sophistication of offensive players, and ever-growing cyber financial gains to offenders, any cyber strategy that relies on constructing a non-vulnerable, weakness-free service in 2022 is clearly too naïve. It seems the only viable strategy is to:

_Admit that your services may be vulnerable._

In other words, consciously accept that you will lose any battle aimed to create non-vulnerable services. If your opponents find a single weakness as an entry-point, you lose. Admitting that although your best efforts, all your services are most likely vulnerable is an important first step. Next, we need to discuss what can you do about it...

_This post focuses on microservices deployed with Kubernetes and points to [Guard](http://knative.dev/security-guard), an open source project designed to help monitor and control your Kubernetes deployed microservices, given that they are vulnerable._  

## Lose the battle - win the war!
Being vulnerable does not necessarily mean you lost the war. Though your services are vulnerable in some ways unknown to you, offenders still need to identify these vulnerabilities and then exploit them to win the war. If offenders fail to exploit your service vulnerabilities, you win! In other words, having a vulnerability that can’t be exploited, represents a risk that can’t be realized.

{{< figure src="Example.svg" alt="Image of an example of offender gaining foothold in a service" class="diagram-large" caption="Figure 1. A Offender gaining foothold in a vulnerable service" >}}

The above diagram shows an example in which the offender does not yet have a foothold in the service. I.e., It is assumed that your service does not run code controlled by the offender on day 1. In our example the service has vulnerabilities in the API exposed to clients. To gain an initial foothold the offender uses a malicious client to try and exploit one of the service API vulnerabilities. The malicious client sends an exploit that triggers some unplanned behavior of the service.

More specifically, let’s assume the service is vulnerable to an SQL injection, the developer failed to do proper sanitization of user input, allowing clients to send values that will change the intended behavior. In our example, if a client sends a query string with key “username” and value of _“tom or 1=1”_, the client will receive the data of all users. Exploiting this vulnerability requires the client to send an irregular string as the value. Note that naive users will not be sending a string with spaces or with the equal sign character as a username, instead they will normally send legal usernames which for example may be defined as a short sequence of characters a-z. No legal username may trigger service unplanned behavior.

In this simple example we can already identify several opportunities to detect and block an attempt to exploit the vulnerability (un)intentionally left behind by the developer, making the vulnerability unexploitable. First, the malicious client behavior differs from the behavior of naive clients, as it sends irregular requests. If such change in behavior is detected and blocked, the exploit will never reach the service. Second, the service behavior in response to the exploit, differs from the service behavior in response to a regular request. The service irregular behavior may include making subsequent irregular calls to other services such as a data store, taking irregular time to respond, and/or responding to the malicious client with an irregular response (e.g., containing much more data then normally sent in case of naive clients making regular requests). Service behavioral changes, if detected, will also allow blocking the exploit in different stages of the exploitation attempt.

More generally:

- Monitoring the behavior of clients can help detect and block exploits against service API vulnerabilities. In fact, deploying an efficient client behavior monitoring makes many vulnerabilities unexploitable and others very hard to achieve. To succeed, the offender needs to create an exploit undetectable from regular naïve requests.

- Monitoring the behavior of services can help detect services as they are being exploited regardless of the attack vector used. Efficient service behavior monitoring limits what an attacker may be able to achieve as the offender need to ensure the service behavior is undetectable from regular service behavior.

Combining both approaches may add a protection layer to the deployed vulnerable services, drastically decreasing the probability for anyone to successfully exploit any of the deployed vulnerable services. Next, lets identify four use cases where you need to use security-behavior monitoring.

## Use cases
One can identify the following four different stages in the life of a any service from a security standpoint. In each stage, security-behavior monitoring is required to meet different challenges:


Service State | Use case | What do you need in order to cope with this use case?
------------- | ------------- | -----------------------------------------
**Normal**   | <u>No known vulnerabilities</u> <br /> The service owner is normally not aware of any known vulnerabilities in the service image or configuration. Yet, it is reasonable to assume that the service has weaknesses. | **Provide generic protection against any unknown, zero-day, service vulnerabilities.** <br /> <br />Detect/block irregular patterns sent as part of incoming client requests that may be used as exploits.
**Vulnerable** | <u>An applicable CVE is published</u> <br /> The service owner is required to release a new non-vulnerable revision of the service. <br /> <br /> Research shows that in practice this process of removing a known vulnerability may take many weeks to accomplish (2 months on average).   |  **Add protection based on the CVE analysis.** <br /> <br />Detect/block incoming requests that include specific patterns that may be used to exploit the discovered vulnerability. Continue to offer services, although the service has a known vulnerability. 
**Exploitable**  | <u>A known exploit is published</u> <br /> The service owner needs a way to filter incoming requests that contain the known exploit.   |  **Add protection based on a known exploit signature.** <br /> <br />Detect/block incoming client requests that carry signatures identifying the exploit. Continue to offer services, although the presence of an exploit.  
**Misused**  | <u>An offender misuses service pods</u> <br /> The offender can follow an attack pattern enabling him/her to misuse pods. The service owner needs to restart any compromised pods while using non compromised pods to continue offering the service. Note that once a pod is restarted, the offender needs to repeat the attack pattern before he/she may again misuse it.  |  **Identify and restart service instances that are being misused.** <br /> <br />At any given time, some service pods may be compromised and misused while others behave as designed. Detect/remove the misused pods while allowing other pods to continue offering  the service.

Next,let's discuss why microservice architecture is well suited to security-behavior monitoring.

## Security-Behavior of Microservices vs Monoliths
Kubernetes is often used to support workloads designed with microservices architecture. By design, microservices aim to follow the UNIX philosophy of "Do One Thing And Do It Well". Each microservice has a bounded context and a clear interface. In other words, we can expect the microservice clients to send relatively regular requests and the microservice to present a relatively regular behavior as a response to these requests. Consequently, microservice architecture is an excellent candidate for security-behavior monitoring.

{{< figure src="Microservices.svg" alt="Image showing why microservices are well suited for security-behavior monitoring" class="diagram-large" caption="Figure 2. Microservices are well suited for security-behavior monitoring" >}}

The diagram above clarifies how dividing a monolithic service to a set of microservices improves our ability to perform security-behavior monitoring and control. In a monolithic service approach, different client requests intertwined, resulting in diminished ability to identity irregular client behaviors. Without prior knowledge, an observer of the intertwined client requests will find it hard to distinguish between types of requests and their related characteristics. Further, internal client requests are not exposed to the observer. Lastly, the aggregated behavior of the monolithic service is a compound of the many different internal behaviors of its components, making it hard to identify irregular service behavior. 

In a microservice environment, each microservice is expected by design to offer a more well-defined service and serve better defined type of requests. This makes it easier for an observer to identify irregular client behavior and irregular service behavior. Further, a microservice design exposes the internal requests and internal services which offer more security-behavior data to identify irregularities by an observer. Overall, this makes the microservice design pattern better suited for security-behavior monitoring and control.

## Security-Behavior Under Kubernetes
Kubernetes deployments seeking to add Security-Behavior may use [Guard](http://knative.dev/security-guard), developed under the CNCF project Knative. **Guard can be installed to protect any HTTP-based Kubernetes service** and is <u>also</u> well integrated into the Knative automation suite that runs on top of Kubernetes.

See:
- [Guard on Github](https://github.com/knative-sandbox/security-guard)
- The Knative automation suite - Read about Knative, as an [Opinionated Kubernetes](https://davidhadas.wordpress.com/2022/08/29/knative-an-opinionated-kubernetes) implementation that simplifies and unifies the way web services are deployed on Kubernetes.
- You can also find more information on how to [secure microservices by monitoring behavior](https://developer.ibm.com/articles/secure-microservices-by-monitoring-behavior/)

The goal of this post is to recruit the Kubernetes community to action and introduce Security-Behavior monitoring and control to help secure Kubernetes based deployments.

1. Analyze the cyber challenges presented for different Kubernetes use cases
1. Add appropriate security documentation for users on how to introduce Security-Behavior monitoring and control.
1. Consider how to integrate with tools that can help users monitor and control their vulnerable services.

***
#### _You are welcome to get involved and join the effort to develop Security-Behavior monitoing and control for Kuberetes; Share feedback and contribute to code, docs, and improvements of any kind._
***