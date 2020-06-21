---
layout: post
title:  "Simulating Systems with Javascript"
date:   2020-06-21 12:00:00 -0400
tags: javascript concepts systems
description: Using ES6 generators to simulate a small system from Thinking in Systems.
---
I just finished reading the wonderful [Thinking in Systems by Donella H. Meadows](https://www.chelseagreen.com/product/thinking-in-systems/). This was a good introductory text for systems thinking, specifically the study of systems (a group of parts with potentially some relationships) and their behaviors over time. The book focused particularly on real-world scenarios - some examples being a bathtub filling with water while draining (potentially different rates), a thermostat and how it is affected by room insulation, and a car dealership trying to understand consumer demand. 

A general language for documenting these systems and their relationships is presented along with some common examples. The following chapters then help guide the reader along how they might try and influence systems by focusing on certain inner system relationships as well as common pitfalls. Conveniently, the book also contains an excellent appendix containing formulas for all the graphs. 

In order to try and wrap my head around the book content, I thought it would be fun to try and write a little simulation for a system. This also lends itself nicely to working with [ES6 iterators / generators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators). I'll preface the rest of this post by stating that I am greatly simplifying the book content and am by no means an expert 🙂. I highly recommend picking it up if this sounds intriguing.

### Our Example System

The exampe we will be modelling involves a car dealership, which has a certain number of cars available on its lot, and its reaction over time to customer demand. We can order more cars to our lot. We want to avoid having too many unsold cars on the old (real estate is expensive!) while also ensuring we always have cars to sell. We'll be looking at how our system evolves using *days* as the time unit.  

As an arbitrary goal, we want to always have enough inventory on hand for **10 days of sales**.

In this simplified example, a system has a **stock** and **flows**. The **stock** is the countable object at the core of the system (in our case, this is the *number of cars on the lot*). The **flows** can be split into *in-flows* and *out-flows*. As time passes, *in-flows* decide how much is added to the stock (eg. deliveries of new cars to the lot) and *out-flows* decide how is removed from the stock (eg. customer purchases a car). A flow is not constant &mdash; these may change over time and can react to the stock value [^1]. Our goal is to observe how stock changes over time based on the flows. 

![Image describing the system with in-flows pointing to stock and stock pointing to out-flows](/assets/images/systems/system.png){:class="centered-image" width="360px"}


The system's **in-flow** is deliveries. We replenish our stock of cars on the lot by placing an order with the factory. From the get go, this introduces an interesting twist: *deliveries take time*. Our system will therefore have a **delivery delay** (ie. number of days for a delivery to arrive) as a configurable setting to explore. 

The system's **out-flow** is sales. This is driven by customer demand. 

This all sounds well and good... but how do we decide the number of cars we want to order at the end of each day? We can start out simply by ordering the number of cars that were sold. But remember that deliveries are not instantaneous. What if we had a large temporary spike in sales, and then a few days later get way more cars than needed for our goal? What if demand plummets? We'll just keep accumulating cars on our lot! We now introduce a second configurable setting to explore: **perception delay**. This is the number of days of sales to look at when deciding how many cars to order. Our strategy could take the average number of sales over the selected days and use that as the amount of cars to order. This dampens out the effect of a temporary spike in either direction of demand.

The concept of change in demand brings up a good point though. What happens if this change is sustained? This means our desired inventory to meet our goal (enough for 10 days of sales) would now change! While we accounted for the next day's perceived sales above, we may want to try adjusting our inventory on top of this to account for the change in desired inventory. This adjustment could be significant, so we'd like to make sure a trend is sticking around before comitting. Ideally, we'll split up our adjustments over a period of days so that we can stop if need be. This introduces the concept of a **response delay**: the number of days to split up adjusting a difference between our inventory and desired inventory.

So at the end of a given day, we will look at the sales averaged over our perception delays. This gives us an idea of what we *perceive* the sales to be for the next 10 days. This gives us our desired amount of cars on the lot. We then compare this with our current inventory to determine the *discrepancy*: the how far off are we (in either direction) from our goal. Combining this, we can then place an order which includes the sales we think will happen the next day as well as the *discrepancy* (can be positive or negative) between our current inventory and what we think we'll need for the next 10 days.

To summarize our system,

- at the end of a given day, our cars on the lot is:  `cars on lot yesterday + deliveries - sales`
- `deliveries` is based on the `orders placed x days ago` where `x = delivery delay`
- `sales = min(customer demand, cars on lot)` as we can only sell what we have
- `perceived sales = average sales for last x days` where `x = perception delay`
- to meet our goal, our `desired inventory` is `perceived sales * 10`
- the `discrepancy` is `desired inventory - current cars on lot`. This factors in our current inventory to make sure we don't over or under-order to meet our goal.
- the orders to place for that day is then `(perceived sales + discrepancy/response delay)`, or 0 (as we can't send back cars). This uses the response delay to help adjust the discrepancy over a set number of days.

We can then analyze how our system behaves based on the following inputs:

- Delays. The delivery, perception and response delays all impact the system. In a real-world system, some of these may be within your control.
- Customer demand. We can try different functions here to see how our system reacts and what pushes it to break.
- An initial set of deliveries. We won't necessarily have enough data to react for the first few days, so we could pre-seed a fixed number of deliveries.

### Translating to Code

In order to plot the data for how a system works given a set of inputs, I'll be using an `ES6 generator`.

For the purposes of our examples, we'll force an initial set of deliveries for the first 5 days of 20 cars.

{% highlight javascript %}
function* generateSystem(perceptionDelay, responseDelay, deliveryDelay) {
  // The initial stock.
  let inventory = 200;
  // Order placed on day [index].
  let orders = [];
  // Actual sales on day [index].
  let sales = [];
  
  for (let day = 0; day < Infinity; day++) {
    // This is our inflow.
    let deliveries;
    // We order 20 cars a day on the first 5 days, and then after that use 
    // the delivery system.
    if (day < 5) {
      deliveries = 20;
    } else {
      // This models the delivery delay.
      deliveries = orders[day - deliveryDelay];
    }
  
    // This is our outflow for the day.
    let demand = getDemand(day)
    sales.push(demand);
    
    // Now we decide how many orders to place for the day. We do this
    // by averaging the sales for the last x days, where x is our
    // perception delay.
    //
    // Calling slice with a negative index gives you the last x 
    // elements.
    let eligibleSales = sales.slice(-perceptionDelay);
    let perceivedSales = 
      eligibleSales.reduce((sum, val) => val += sum) / 
        eligibleSales.length;
          
    // Our goal is to have enough inventory for the next 10 days
    // worth of sales.
    let desiredInventory = perceivedSales * 10;
    let discrepancy = desiredInventory - inventory;
    
    // Our order includes the perceived sales and also tries to fix
    // the discrepancy.
    let order = 
      Math.max(
        Math.round(perceivedSales + discrepancy / responseDelay), 0);
    orders.push(order);
   
    // Update our stock and return the value for plotting.
    inventory = inventory + deliveries - demand;
    yield inventory;
  }
}
{% endhighlight %}

Finally, we have to configure customer demand. I extracted a parameterized function here based on the day so you could try modelling with different settings. As a starting point, let's pretend customers normally buy 20 cars and we then have a 10% increase in customer demand at day 25.


{% highlight javascript %}
function getDemand(day) {
  if (day < 25) {
    return 20;
  } else {
    return 22;
  }
}
{% endhighlight %}

### Simulation

Here is the end result of the simulation, where we see how stock changes over time. Play around with the variables - see what you find. I really enjoyed just seeing how whacky this could get. 

{% include simulation.html %}

### Conclusion

I find the oscillations that arise from shortening perception and response delay particularly interesting. This felt almost counter-intuitive at first, since it feels like taking on a risk - "what if this spike keeps going and I miss out?". We can also see the drastic impact supply chain can have, and this gives an understanding as to why improvements in logistics can greatly improve efficiency.

As I initially stated, this is a very simple model. The book touches on how models abstract away significant parts of reality and are just an approximation. These delays aren't known, static values. We can't always simply improve our response time. Parts of our system that we thought were simple can actually be deeply nested sub-systems. We could be completely wrong about relationships in a system. This can quickly become a deep rabbit-hole. But it sure is fun to observe the simulation run. :) 

### Footnotes

[^1]: This is just one kind of loop, and can be a balancing loop or a feedback loop. The book covers this in much greater detail. An example to think here involves a thermostat. The stock in this scenario is the room temperature. An in-flow would be heat controlled by a thermostat. The thermostat may turn itself on/off based on the room temperature (ie. the flow reacts to the stock).



The scenario we will be modelling here involves a car dealership trying to understand consumer demand and 