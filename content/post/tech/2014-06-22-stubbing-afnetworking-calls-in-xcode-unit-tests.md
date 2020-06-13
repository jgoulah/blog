---
title: Stubbing AFNetworking Calls In XCode Unit Tests
author: jgoulah
type: post
date: 2014-06-22T16:58:52+00:00
url: /2014/06/stubbing-afnetworking-calls-in-xcode-unit-tests/
categories:
  - iOS
tags:
  - afnetworking
  - ios
  - mobile
  - ohhttpstubs
  - stubs
  - xcode

---
## Intro

A <a href="http://en.wikipedia.org/wiki/Unit_testing" title="unit testing" target="_blank">unit test</a> by definition of testing the smallest possible part of the application, should eliminate all dependencies from the systems its interacting with. This in turn will remove any unknown or variable outcomes from the tests, focusing on the isolating the specific code you&#8217;re interested in. In the case of networking calls, it is unideal for tests that deal with the (un)marshalling of data to depend on the network. Therefore a stubbing library is needed. 

## OHHTTPStubs to the Rescue!

Luckily there is a great library to do just what we need here &#8211; stubbing out some network calls &#8211; called <a href="https://github.com/AliSoftware/OHHTTPStubs" title="OHHTTPStubs" target="_blank">OHHTTPStubs</a>. 

Because these tests are asynchronous, the network requests are sent in a separate thread or queue different from the thread used to execute the test case. So you&#8217;ll have to ensure that your unit tests wait for the requests to finish and their response arrived before performing the assertions in your test cases. Otherwise your code making the assertions will execute before the request had time to get its response.

OHHTTPStubs includes a custom _AsyncSenTestCase_ subclass of _SenTestCase_ that provides methods like _waitForAsyncOperationWithTimeout:_ and _notifyAsyncOperationDone_. We can utilize these methods in our test cases.

## A Networking Stub

Most people are using the <a href="https://github.com/AFNetworking/AFNetworking" title="AFNetworking" target="_blank">AFNetworking</a> library so this example will show a test using that.

In this case I&#8217;ve created a network wrapper that allows turning stubs on and off. The request here happens to be mocking XML data, but JSON follows the same pattern. This is an implementation of a class called GrillNet that has two functions:



The two functions are: _stubsOn_ to activate the stubbed networks calls, and _getGrillStatusWithURL_ which invokes a network request. If we call stubsOn before the getGrillStatusWithURL call is made, it will invoke the stub and return the mock data which I&#8217;ve generated and put into the file _cyberq.xml_. 

Notice that _getGrillStatusWithURL_ implements a success and failure callback. Thus I&#8217;ve created a protocol to respond to the two scenarios which asks for optional implementations of _notifyDone_ and _notifyDoneWithData:data_, shown here:



Our tests can implement this protocol to assist with the notifying that our asynchronous operation is done by calling _notifyAsyncOperationDone_. The class _GrillzTests_ subclasses _AsyncSenTestCase_ and implements _GrillNetActionProtocol_.



And in this same class we can implement _notifyDone_ and _notifyDoneWithData:data_. They will both call _notifyAsyncOperationDone_ from _AsyncSenTestCase_ but _notifyDone_ is only called on failure when no data is returned, while _notifyDoneWithData:data_ is invoked when the request is successful . 



Therefore it is the job of _notifyDoneWithData:data_ to do something with this data. In this case we are setting the values in a _GrillStatus_ object that was used in the test assertion comparing the value in __gs_ above.
  


## References

<a href="https://github.com/AliSoftware/OHHTTPStubs" title="OHHTTPStubs" target="_blank">https://github.com/AliSoftware/OHHTTPStubs</a>

<a href="https://github.com/AFNetworking/AFNetworking" title="AFNetworking" target="_blank">https://github.com/AFNetworking/AFNetworking</a>

<a href="https://github.com/AliSoftware/OHHTTPStubs/wiki/OHHTTPStubs-and-asynchronous-tests" target="_blank">https://github.com/AliSoftware/OHHTTPStubs/wiki/OHHTTPStubs-and-asynchronous-tests</a>

<a href="http://andreamazz.github.io/blog/2014/01/10/network-stubs-with-ohhttpstubs/" target="_blank">http://andreamazz.github.io/blog/2014/01/10/network-stubs-with-ohhttpstubs/</a>

<a href="https://github.com/AliSoftware/OHHTTPStubs/wiki/Usage-Examples" target="_blank">https://github.com/AliSoftware/OHHTTPStubs/wiki/Usage-Examples</a>
