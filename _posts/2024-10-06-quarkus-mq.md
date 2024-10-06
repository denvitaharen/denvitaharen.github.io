---
layout: post
title:  "Quarkus and messaging with IBM MQ"
date:   2024-10-06 21:31:17 +0200
categories: quarkus java jms
---
### _tl;rd_

_This is an summary of my investigations. It was done on the late summer of 2024 and it based on Quarkus 3.8.6._

_If AMQP is enabled on your MQ host, then check out [SmallRye Reactive AMQP](https://quarkus.io/guides/amqp) as it seams like an really nice alternative. If AMQP isn't enabled, then you have to fall back to the JMS protocol and then Apache Camel is the best option._

When upgrading an existing microservice written with Quarkus 2.16 to 3.8, I ran in with a problem regarding JMS and consuming messages from an IBM MQ queue. The existing implementation used an I would say traditional JMS implementation with creating an `MessageListner` and the solution worked quite good. It did what it was suppose to do, but we had some problems with REST clients and the dev services didnt work as smoth as the rest of Quarkus did. We found some workarounds for this and 

When we upgraded to Quarkus 3.8 and also changed to preferred RESTEasy Reactive framework, we got an problem that no REST clients worked. After some testing we found out that the problem didnt come with the upgrade to Quarkus 3.8 but with the upgrade to RESTEasy Reactive. When we went back to RESTEasy Classic it started working again. The problem we faced where that when we injected the restclients, we got an `ClassNotFoundException` on the restclient.

We did some investigation about it and from what I found was that the `MessagerListner` from the `IBM AllClient jar` started the `MessageListner` in an sepreate thread from Quarkus and it was rather lucky that we got it working earlier.

Why did it stop working with RESTEasy Reactive? I don't have any proof for it, but I belive that RESTEasy Reactive is an rewrite and is written in a way that its more integratad in Vertx, the base of Quarkus. This change in framework is thought to make the RESTClients not avalible in the thread of the `MessageListner`. Quarkus doesn't have any official support for JMS, so our old solution was not supported and the only supported alternatives where based on extensions.

Based on my research I identified these alternatives:
- AMQP
- SmallRye Reactive JMS
- Ironjacamar and custom JCA resource adapter
- Apache Camel

## AMQP
The AMQP interface looks nice, especially if you use the [SmallRye Reactive AMQP](https://quarkus.io/guides/amqp) but the AMQP wasn't activated on our MQ instances, so this wasn't an alternative for us.

## SmallRye Reactive JMS
When I searched the internet for diffrent alternatives, I found some had used the SmallRye Reactive JMS extension, for instance this comment on [StackOverflow](https://stackoverflow.com/a/77625283). The code is really clean, it removed a lot boiler plate and it worked straight as it should, the dev services etc worked like a charm.

Straight away I found one problem and that was that a big no no and that was that if something when wrong in the code and I throwed an exception to nake the massage fall to the backout queue (error queue), the message disapered. The documentation didn't mention anything about it and I tried diffrent sessions mode, diffrent exceptions but I didnt really get it to work as it does with the `MessageListner`. The errorhandling isn't mention in the documentation and when searching on the internet, it seams that the reactive JMS doesnt have this function when I searched this (late summer 2024).

An other problem is that many parts of the SmallRye Reactive package is officially supported in Quarkus, the SmallRye Reative JMS isnt. But loosing messages isn't an acceptable solution.

## Ironjacamar and custom JCA resource adapter
When looking at alternatives, I looked at the [Quarkus Artemis Resource Adapter](https://docs.quarkiverse.io/quarkus-artemis/dev/quarkus-artemis-ra.html), I found that the someone hade made it possible to connect to an Artemis queue with an MessageListner.

So I set out to make my own JCA resource adapter and I got it to work in form of connection to the queue, consuming messages and correct errorhandling, but I still got the problem that it seamed that it was still running in an seprate thread controlled by the IBM code and we still didn't get the restclients to work. So this wasn't solving the problem.

## Apache Camel
My last resort was Apache Camel, an extension I saw early but I discarded due to it felt overwhelming to learn it and understand it. But now it seamed that it was my last hope. There is three diffrent extensions for JMS and Apache Camel, but I choose the [JMS Extension.](https://camel.apache.org/camel-quarkus/3.15.x/reference/extensions/jms.html).

I just followed the documentation for the JMS extension and after some tries I got it to connect to my queue and consume messages. Since Apache Camel is fully integrated in Quarkus, I didn't fall in to any problems with restclients and it work just like it suppose too. Then trying out the errorhandling it also worked as it should, with the message heading to the errorqueue as it should. And since it integrated as it should, it means that you can post and message, consume it, make an change in the code, post and new message and Quarkus will reload the code before it consumes the message, just like it does when testing an REST endpoint.

An other nice feature is that if you want health and metrics on your integrations, then in typical Quarkus fascion, its just to at the `MicroProfile Health` and the `Micrometer` and of you go. No need to write it your self.

An other big advantage is that Apache Camel has many features, so if you want to throttle how many messages you consume, there is an [extension](https://camel.apache.org/components/4.8.x/eips/throttle-eip.html) for that. The list of extensions at Camels homepage is huge.

The only downside I have found about using Apache Camel is that since its an complete integration framework, it has capable of doing a lot of stuff and at first glance, it feels like its really much to learn and there is even full [books](https://www.manning.com/books/camel-in-action-second-edition) on mastering Camel, all just for integration with a JMS. But for the stuff that I have done, it isn't so hard to learn.