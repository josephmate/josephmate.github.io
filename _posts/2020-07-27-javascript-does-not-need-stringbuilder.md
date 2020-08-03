---
layout: post
title: "Javascript Does Not Need a StringBuilder"
date: 2020-07-27 20:14
author: joseph.mate
comments: true
categories: [Java, Javascript, StringBuilder]
---

Today I learned that Javascript does not need a StringBuilder for accumulating a large number of concatinations.
As a Java programmer, that came as a shock to me.

# Effective Java
A StringBuilder is grilled into Java programmers. Effective Java says, "Item 51: Beware the performance of string concatenation."
Many books, articles, blog posts and stack overflow questions have extensively covered this in detail 
[2](http://www.thefinestartist.com/effective-java/51)
[3](http://jtechies.blogspot.com/2012/07/item-51-beware-performance-of-string.html)
[4](https://www.corejavaguru.com/effective-java/items/51)
[5](https://dzone.com/articles/string-concatenation-performacne-improvement-in-ja)
[6](https://stackoverflow.com/questions/4645020/when-to-use-stringbuilder-in-java)
[7](https://stackoverflow.com/questions/10078912/best-practices-performance-mixing-stringbuilder-append-with-string-concat)
[8](https://stackoverflow.com/questions/1532461/stringbuilder-vs-string-concatenation-in-tostring-in-java).

Now you can see why it came as such a shock to me.
I assumed Javascript would do the same thing as Java to maintain the immutability of strings.
Java allocates a new array containing the result of each successive concatenation resulting in the O(N^2) runtime complexity.

# Experiments
When I first learned this from a co-worker, I couldn't believe it!
I put together some experiments to confirm.
I measured how long it would take to do different powers of 2 of concatenations in Java and Javascript.
I am not interested in a comparison between the languages.
I am curious about how runtime grows as the number of concatenations grows.

If you would like to checkout a copy of the source code take a look at the last two sections (Java source and Javascript source) of this page or check [my repo](https://github.com/josephmate/JavaVsJavascriptStringBuilder).

You can try out the Javascript experiment in your browser at my [git hub page](https://josephmate.github.io/JavaVsJavascriptStringBuilder/).


I've summarized how long each number of concats took in the table below.
I looked at using naive += String concatenation, Java's StringBuilder, and Javascript's Array.join.

|           |               |                    | Chrome          | Chrome          | Firefox         | Firefox         | NodeJS          | NodeJS          |
|           | Java          | Java               | Javascript      | Javascript      | Javascript      | Javascript      | Javascript      | Javascript      |
| # concats | naive += (ms) | StringBuilder (ms) | naive += (ms)   | Array.join (ms) | naive += (ms)   | Array.join (ms) | naive += (ms)   | Array.join (ms) |
|-----------|---------------|--------------------|-----------------|-----------------|-----------------|-----------------|-----------------|-----------------|
|      2^10 |             4 |                  0 |               0 |               0 |               0 |               0 |               1 |               0 |
|      2^11 |            12 |                  0 |               0 |               0 |               0 |               0 |               1 |               0 |
|      2^12 |            28 |                  0 |               1 |               0 |               0 |               0 |               1 |               1 |
|      2^13 |            99 |                  0 |               1 |               1 |               1 |               0 |               1 |               0 |
|      2^14 |           374 |                  1 |               3 |               2 |               1 |               0 |               1 |               2 |
|      2^15 |          1197 |                  2 |               2 |               2 |               2 |               1 |               4 |               4 |
|      2^16 |          2961 |                  1 |               8 |               4 |               3 |               3 |              10 |               6 |
|      2^17 |          4505 |                  3 |              15 |               8 |               6 |               4 |              29 |              12 |
|      2^18 |         21945 |                  5 |              30 |              18 |              14 |              11 |              29 |              22 |
|      2^19 |         94077 |                  6 |              79 |              37 |              12 |              18 |             112 |              54 |
|      2^20 |        412675 |                 11 |             144 |              71 |              42 |              44 |             206 |              93 |
|      2^21 |       1886317 |                 39 |             408 |             156 |              68 |              70 |             460 |             245 |
|      2^22 |       8529431 |                 70 |             906 |             318 |             172 |             149 |             813 |             425 |
|      2^23 |      TOO LONG |                119 |            1650 |             621 |             272 |             384 |            2233 |             753 |
|      2^24 |      TOO LONG |                209 |            3418 |            1416 |             518 |             766 |            3599 |            1920 |
|      2^25 |      TOO LONG |                492 |            7411 |            3275 |            1836 |            1846 |            6384 |            4048 |
|      2^26 |      TOO LONG |                898 |           14548 |            5385 |            5075 |            3655 |           13266 |           20333 |
|      2^27 |      TOO LONG |               1738 |             OOM |             OOM |            9850 |           10912 |           33434 |             OOM |

At 2^23, the naive Java String concatenation took long to run.
I gave up at 2^22 when it took over 2 hours!
Each doubling of the input size results in more than double the runtime, as we expected because of the O(N^2) complexity.
However, with Javascript it's hard to tell.
The growth looks kind of linear but maybe log linear.
Lets investigate with a plot.
We won't plot the naive Java concatenation since it grows too quickly and the scale would make it difficult to analyze the growth of the remaining methods.

<script src="https://d3js.org/d3.v4.js"></script>
<!-- Color Scale -->
<script src="https://d3js.org/d3-scale-chromatic.v1.min.js"></script>

<div id="chart"></div>

<script>

// https://stackoverflow.com/questions/448981/which-characters-are-valid-in-css-class-names-selectors
// begin with underscore (_), a hyphen (-), or a letter(a–z),
// followed by any number of hyphens, underscores, letters, or numbers
// if the first character is a hyphen, the second character must2 be a letter or underscore
// name must be at least 2 characters long
function toClassName(humanReadableString) {
  return humanReadableString
    .replace("#", "")
    .replace(" ", "")
    .replace("+", "")
    .replace("=", "")
    .replace("(", "")
    .replace(")", "")
    .replace(".", "")
     ;
}

function isNumber(value) {
  return Number.isFinite(value);
}

function lastNumberPoint(dataPoints) {
  for (var i = dataPoints.length - 1; i >= 0; i--) {
    if (isNumber(dataPoints[i].value)) {
      return dataPoints[i];
    }
  }
  return dataPoints[dataPoints.length - 1];
}

// based on
// https://www.d3-graph-gallery.com/graph/connectedscatter_legend.html
// 1. modified the legend to display vertically since there's more lines
// and more text
// 2. sanitize column names for use in CSS class selectors
// 3. last value is missing, so use a loop to find the last value instead
var margin = {top: 10, right: 100, bottom: 30, left: 100},
    width = 700 - margin.left - margin.right,
    height = 700 - margin.top - margin.bottom;

var svg = d3.select("#chart")
  .append("svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", height + margin.top + margin.bottom)
  .append("g")
    .attr("transform",
          "translate(" + margin.left + "," + margin.top + ")");


//Read the data
d3.csv("/data/2020-07-28_experiment_data.csv", function(data) {

    // List of groups (here I have one group per column)
    var allGroup = [
      "Java8 StringBuilder (ms)",
      "Chrome Javascript naive += (ms)",
      "Chrome Javascript Array.join (ms)",
      "Firefox Javascript naive += (ms)",
      "Firefox Javascript Array.join (ms)",
      "NodeJS Javascript naive += (ms)",
      "NodeJS Javascript Array.join (ms)"
]

    // Reformat the data: we need an array of arrays of {x, y} tuples
    var dataReady = allGroup.map( function(grpName) { // .map allows to do something for each element of the list
      return {
        name: grpName,
        values: data.map(function(d) {
          return {time: d["# concats"], value: +d[grpName]};
        })
      };
    });
    // I strongly advise to have a look to dataReady with
    console.log(dataReady)

    // A color scale: one color for each group
    var myColor = d3.scaleOrdinal()
      .domain(allGroup)
      .range(d3.schemeSet2);

    // Add X axis --> it is a date format
    var x = d3.scaleLinear()
      .domain([0,134217728])
      .range([ 0, width ]);
    svg.append("g")
      .attr("transform", "translate(0," + height + ")")
      // https://observablehq.com/@d3/axis-ticks
      .call(d3.axisBottom(x).ticks(10, "s"));

    // Add Y axis
    var y = d3.scaleLinear()
      .domain( [0,33434])
      .range([ height, 0 ]);
    svg.append("g")
      .call(d3.axisLeft(y));

    // Add the lines
    var line = d3.line()
      .x(function(d) { return x(+d.time) })
      .y(function(d) { return y(+d.value) })
    svg.selectAll("myLines")
      .data(dataReady)
      .enter()
      .append("path")
        .attr("class", function(d){ return toClassName(d.name) })
        .attr("d", function(d){ return line(d.values) } )
        .attr("stroke", function(d){ return myColor(d.name) })
        .style("stroke-width", 4)
        .style("fill", "none")

    // Add the points
    svg
      // First we need to enter in a group
      .selectAll("myDots")
      .data(dataReady)
      .enter()
        .append('g')
        .style("fill", function(d){ return myColor(d.name) })
        .attr("class", function(d){ return toClassName(d.name) })
      // Second we need to enter in the 'values' part of this group
      .selectAll("myPoints")
      .data(function(d){ return d.values })
      .enter()
      .append("circle")
        .attr("cx", function(d) { return x(d.time) } )
        .attr("cy", function(d) { return y(d.value) } )
        .attr("r", 5)
        .attr("stroke", "white")

    // Add a label at the end of each line
    svg
      .selectAll("myLabels")
      .data(dataReady)
      .enter()
        .append('g')
        .append("text")
          .attr("class", function(d){ return toClassName(d.name) })
          .datum(function(d) { return {name: d.name, value: lastNumberPoint(d.values)}; }) // keep only the last value of each time series
          .attr("transform", function(d) { 
            if ( d.value.value > 30000) {
            return "translate(" + x(d.value.time-30000000) + "," + y(d.value.value-1000) + ")"; 
            }
            return "translate(" + x(d.value.time-30000000) + "," + y(d.value.value+1000) + ")"; 
            }) // Put the text at the position of the last point
          .attr("x", 12) // shift the text a bit more right
          .text(function(d) { return d.name.replace(" (ms)", ""); })
          .style("fill", function(d){ return myColor(d.name) })
          .style("font-size", 15)

    // Add a legend (interactive)
    svg
      .selectAll("myLegend")
      .data(dataReady)
      .enter()
        .append('g')
        .append("text")
          .attr('x', function(d,i){ return 30 })
          .attr('y', function(d,i){ return 30 + i*20})
          .text(function(d) { return d.name; })
          .style("fill", function(d){ return myColor(d.name) })
          .style("font-size", 15)
        .on("click", function(d){
          // is the element currently visible ?
          currentOpacity = d3.selectAll("." + toClassName(d.name)).style("opacity")
          // Change the opacity: from 0 to 1 or from 1 to 0
          d3.selectAll("." + toClassName(d.name)).transition().style("opacity", currentOpacity == 1 ? 0:1)

        })
});
</script>

After plotting, the growth looks linear for all of these variations.
However, it's best to keep reading for a bit of complexity analysis.

# Why Javascript Does not Need a StringBuilder

Following the experiments, I wanted to know why.
My research led me to [this email chain](https://www.mail-archive.com/es-discuss@mozilla.org/msg10071.html) where Dr. Axel Rauschmayer
proposes adding StringBuilder to the next version of ECMAScript,
> Is this worthy of ES.next support? Or does it belong into a library?
> 
> The two concatenation approaches I know of are:
> 1. via +=
> 2. push() into an array, join() it after the last push()
> 
> (1) can’t possibly be efficient, but if (2) is OK on all(!) platforms, then a 
> library would be OK. However, given how frequently this is used, I would like 
> this to become part of the standard library.

[Tom Schuster responds](https://www.mail-archive.com/es-discuss@mozilla.org/msg10129.html) that it's not necessary.
> (1) is in  fact really good optimized in modern engines.  (In case you
are interested search for "Ropes: an alternative to strings")
> 
> I think today it's not a very good idea to propose methods on probably
existing performance problems.

I did more digging and learned the ropes are a datastructure the allow immutable String concatenations in O(lgN) time.

[Firefox's Javascript VM's sourcecode](https://hg.mozilla.org/mozilla-central/file/tip/js/src/vm/StringType.h#l73) comments on what Tom Schuster was saying the email chain.

>    To avoid O(n^2) char buffer copying, a "rope" node (JSRope) can be created
>    to represent a delayed string concatenation. Concatenation (called
>    flattening) is performed if and when a linear char array is requested. In
>    general, ropes form a binary dag whose internal nodes are JSRope string
>    headers with no associated char array and whose leaf nodes are linear
>    strings.

Details about the [the rope data structure](https://en.wikipedia.org/wiki/Rope_(data_structure)) are available on Wikipedia.
Concetenation takes O(lgN) time where N is the number of concats in the rope so far.
As a result, to concat N times, the complexity is O( sum from 1 to N (lg N)) which is O(N lg N ) as explained in this [Stackoverflow article](https://stackoverflow.com/questions/38849052/time-complexity-with-log-in-loop).
It's not as good as using a StringBuilder for really large data, which amoritzes to O(N)
However, for the number of concatenations that will work in the browser (2^27), the rope is good enough because behaves linearly until then.
However, if you plan to need more than that, you better use a builder.

# Javascript StringBulder

I tried implementing my own StringBuilder for fun to see if I could improve upon the 10 seconds it takes in Firefox and failed.
My version of StringBuilder implemented in Javascript in Firefox took over 70 seconds to handle 2^27 concatenations.
Javascript provides a couple ways of creating strings directly from arrays.

The first I tried was `String.fromCharCode`.
This method looked like it would be faster; however, at 2^17 it fails with `Uncaught RangeError: Maximum call stack size exceeded`
because `String.fromCharCode` use function parameter arguments.
As a result, when you use `...` array expansion or `Function.apply`, you blow up your stack.

In my second attempt, I tried using [TextDecoder](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder) and [TextEncoder](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder).
In Firefox took over 70 seconds to handle 2^27 concatenations! As a result, you cannot yet implement a StringBuilder that's as efficient as Java within Javascript.

I was able to improve the performance for 2^27 concatenations by using Rust's String which acts similarly to the Rust and connecting to Javascript through WASM.
It only brought it down to 23 seconds, which is still worse than Javascript's += and Array.join. 

# Discussion
Could Java implement a similar optimization to the JVM so that people who overlook this mistake aren't hit with an O(N^2) but a O(NlgN) algorithm instead?

Are ropes good enough? At 2^24 to 2^27 we're measuring the concatenations in seconds and the growth appears linear until that point even though asymptotically it is not.
You could argue that's good enough because that's the memory limit of the browser.

Would a StringBuilder implemented in WASM perform significantly better than += and Array.join if there was a sufficient amount of concatenations?
Unfortunately, I was limited to 2^27 concatenations before running out of memory.
If was able to exceed that memory limit, I expect that WASM, and even my Javascript StringBuilder to eventually perform better due to the O(N) vs. O(NlgN) complexity.

Finally, would we see a performance improvement if we implemented a StringBuilder in the javascript virtual machine instead of JSRope and rebuilt the browser?

# Java source

```java
public static void main(String [] args) {
    final int base;
    final int startPower;
    final int concatPowerLimit;
    final int powerLimit;

    if(args.length >= 4) {
        base = Integer.parseInt(args[0]);
        startPower = Integer.parseInt(args[1]);
        concatPowerLimit = Integer.parseInt(args[2]);
        powerLimit = Integer.parseInt(args[3]);
    } else {
        base = 2;
        startPower = 1;
        concatPowerLimit = 21; // at least 2 hours to compute the 22nd one
        powerLimit = 27;
    }

    for(int power = startPower; power <= 27; power++) {
        int size = (int)Math.pow(2, power);
        runStringBuilderExperiment(base, power, size);
        if (power <= concatPowerLimit) {
            runConcatExperiment(base, power, size);
        }
    }
}

public static void runConcatExperiment(int base, int power, int size) {
    String result = "";
    long start = System.currentTimeMillis();
    for (int i = 0; i < size; i++) {
        result += (i%10);
    }
    long end = System.currentTimeMillis();
    long duration = end - start;
    System.out.println(String.format("concat %d^%d %d %d %d", base, power, size, result.length(), duration));
}

public static void runStringBuilderExperiment(int base, int power, int size) {
    StringBuilder builder = new StringBuilder();
    long start = System.currentTimeMillis();
    for (int i = 0; i < size; i++) {
        builder.append(i%10);
    }
    String result = builder.toString();
    long end = System.currentTimeMillis();
    long duration = end - start;
    System.out.println(String.format("builder %d^%d %d %d %d", base, power, size, result.length(), duration));
}
```

# Javascript Naive Concat and Array.join Source

```javascript
function runConcatExperiment(size) {
    var result = "";
    var start = new Date().getTime();
    for (var i = 0; i < size; i++) {
        result += (i%10);
    }
    var end = new Date().getTime();
    var duration = end - start;
    console.log("concat %d %d %d", size, result.length, duration);
    return duration;
}

function runArrayJoinExperiment(size) {
    var builder = [];
    var start = new Date().getTime();
    for (var i = 0; i < size; i++) {
        builder.push(i%10);
    }
    var result = builder.join();
    var end = new Date().getTime();
    var duration = end - start;
    console.log("concat %d %d %d", size, result.length, duration);
    return duration;
}

function addRow(base, power, size, concatDuration, arrayJoinDuration) {
    var resultsTable = document.getElementById("results");
    var tableRow = document.createElement("tr");
    var powerCell = document.createElement("td");
    powerCell.innerText = base + "^" + power;
    tableRow.append(powerCell);
    var sizeCell = document.createElement("td");
    sizeCell.innerText = size;
    tableRow.append(sizeCell);
    var concatCell  = document.createElement("td");
    concatCell.innerText = concatDuration;
    tableRow.append(concatCell);
    var joinCell  = document.createElement("td");
    joinCell.innerText = arrayJoinDuration;
    tableRow.append(joinCell);
    resultsTable.appendChild(tableRow);
}

function runExperiment() {
    var baseInput = document.getElementById("base");
    var base = baseInput.value;
    var powerLimitInput = document.getElementById("powerLimit");
    var powerLimit = powerLimitInput.value;
    for(var power = 1; power <= powerLimit; power++) {
        var size = Math.pow(2, power);
        var concatDuration = runConcatExperiment(size);
        var arrayJoinDuration = runArrayJoinExperiment(size);
        addRow(base, power, size, concatDuration, arrayJoinDuration);
    }
}
```

# Javascript String Builder
```javascript
function StringBuilder() {

    // cannot use String.fromCharCode due to
    // Uncaught RangeError: Maximum call stack size exceeded
    // so we are stuck with TextEncoder and TextDecoder
    // https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder
    // https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder
    let utf8Decoder = new TextDecoder();
    let utf8Encoder = new TextEncoder();

    this.bufferConsumed = 0;
    this.capacity = 128;
    this.buffer = new Uint8Array(128);

    this.append = function (strToAdd) {
        // O(N) copy but ammortized to O(1) over all concats
        var encodedStr = utf8Encoder.encode(strToAdd);
        while (encodedStr.length + this.bufferConsumed > this.capacity) {
            var tmpBuffer = this.buffer;
            this.buffer = new Uint8Array(this.capacity*2);
            this.capacity = this.capacity*2;
            for(var i = 0; i < this.bufferConsumed; i++) {
                this.buffer[i] = tmpBuffer[i];
            }
        }

        // add the characters to the end
        for(var i = 0; i < encodedStr.length; i++) {
            this.buffer[i+this.bufferConsumed] = encodedStr[i];
        }
        this.bufferConsumed += encodedStr.length;

        return this;
    }

    this.build = function() {
        return utf8Decoder.decode(this.buffer.slice(0, this.bufferConsumed));
    }
}
```

# Rust WASM String
```
mod utils;

use wasm_bindgen::prelude::*;

// When the `wee_alloc` feature is enabled, use `wee_alloc` as the global
// allocator.
#[cfg(feature = "wee_alloc")]
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;

#[wasm_bindgen]
pub struct RustStringBuilder {
    // String push_str uses a vector internally
    // vector amortizes push to O(1) over many appends
    internal: String,
}

#[wasm_bindgen]
impl RustStringBuilder {

    #[wasm_bindgen(constructor)]
    pub fn new() -> RustStringBuilder {
        RustStringBuilder { internal: String::from("") }
    }

    pub fn append(&mut self, to_append: String) {
        self.internal.push_str(to_append.as_str())
    }

    pub fn build(&self) -> String  {
        String::from(self.internal.as_str())
    }
}
```