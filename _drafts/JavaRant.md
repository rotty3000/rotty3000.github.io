---
title: Java Rant
excerpt: A rant about Java and the JVM.
---


 I'll try to get into that in a minute.

And costs are a big deal. In fact they are the biggest deal with talking about Cloud Native applications. Because rather than paying for bare metal hardware, I need to think about the cost of compute and storage that I don't own at a granular level.

But how does this massive shift in infrastructure requirements affect the codebase?

Well it's a question that comes down to balancing "density" and "elasticity". Let me explain.

In order to reduce costs in Cloud Native applications you need **high density**. You need to squeeze as much value as possible out of as few resources as possible. So a model where compute is delivered just in time as a response to demand is a good thing. Ok, that's a given.

The other side of the coin is **elasticity**. You need to be able to scale up and down as demand changes; a.k.a. "elasticity". This is where the rubber meets the road. If your application only has a single dimension of scale, then you have only that one lever to pull. But if you have multiple dimensions of scale, then you have multiple levers to pull. The more levers the more elastic you can be, but the more complex your application becomes.

Allow me to use an analogy to explain.

Suppose you own a moving company. And suppose the only boxes you have available are the size of a refrigerator. If everything you are going to move is the size of a refrigerator, then you are good. However you will need several strong individuals and maybe special equipment to move them. It will be expensive in labour and equipment. Scaling your business will depend on how many strong individuals you can hire and how much equipment you can afford.

On the other hand, if the only boxes you have available are the size of a hair brush, first, you'll need a lot of them, secondly, you'll be able to hire a lot of average people to move them and you won't need any special equipment, though some items might need to be chopped up to fit and then reassembled at the other end. There will be significant time spent packaging and moving all those little boxes. Scaling your business shouldn't be a problem except for the logistics of all the chopping and rebuilding and moving all those little boxes.

You see where I'm going with this. You probably want a variety of boxes at your disposal and you probably want to scale your work force dynamically as demand changes. But having boxes of different sizes and shapes means you now need to optimize how they are moved and stored.

It essentially boils down to a high tech game of tetris.

Your application should be able to scale in different directions. At one moment in time it's about fitting more features into less space and at another point in time it's about taking "some" of those features and giving them more space.

Within the Cloud Native ecosystem, the refrigerator box is the Virtual Machine and the hair brush box is the Function as a Service (FaaS) model. You don't want to cut yourself off from either end of the spectrum. But if your application can only essentially exist as a VM then you're going to have a bad time.

So how did all this affect me as a Java developer? Well, working in the Cloud Native ecosystem forces you to think about the "elasticity" side of the equation. At every turn you're faced with the question of how can we reduce costs?

The first step is to increase density.

Build a multi-tenant monolith! Problem solved!

But wait, we have an upgrade problem! Ok, let's shard the database! Problem solved!

But wait, we're seeing the noisy neighbour problem! Ok, let's separate into microservices! Problem solved!

But wait, customer's want auto scaling? Let's add more nodes to the cluster!

But wait, we're seeing the cost problem? Ok, let's build a FaaS! Problem solved!

But wait, we're seeing the cold start problem? Ok, let's aggregate back into microservices!



















