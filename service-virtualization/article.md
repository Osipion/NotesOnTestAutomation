# WireMock vs HoverFly vs Canned
### Stubbing, Mocking and "Service Virtalization" in System Test Automation

*Draft 2017-02-19*

Mocking and stubbing are increasingly common terms in the world of iterative
development. These tools all offer strategies for isolating the parts of a system
we are interested in from the complexity/unpredictability of other system
components.

There's a growth in these tools of late - [countless][ninject] [mocking][rhino] and [DI][autofac] [frameworks][castlew]
abound in the lands of JVM and .NET, and new vendors are springing up offering
service virtualization tools as alternatives to mocking. Most applications now
seem to involve IPC, usually over HTTP, and vendors hope to take advantage of
the general trend back towards fine-grained Service Oriented Architectures. In
trying to evaluate these tools, I found myself trying to work out on what
criteria such a tool be should even be assessed. So before I compare three
different approaches to faking HTTP dependencies, I'm going to try to lay out a
basic approach to breaking down what faking technology can do for QA.

Since in practice the terms "mock" and "stub" are often used
interchangably, for now I will use the term "faking" to refer to all strategies
which involve replacing production components with fake versions of those
components during testing.

## Are you **S**OL**ID**?

Almost all effective faking strategies are enabled by good software design.
Whether it is at a class/module level, or at the level of large system
components (e.g. databases, file systems, APIs, system clocks/synchronization
services etc.), single responsibility, interface segregation and dependency
inversion are what enables this.

Most developers these days are likely familiar with at least one dependency
injection framework. To allow a developer to replace an actual MongoDB
connection with a fake one, instead of the class under test constructing the
connection itself, an instance of the connection is injected into it.
Dependency injection frameworks are ways of achieving dependency inversion - of
having software elements depend upon abstractions that can be swapped in and
out without recompilation. To make it easy to swap out these services, the class
under test should depend on the most abstract form of the service possible.
Interface segregation makes this easier, by reducing the amount of behaviour that
needs to be implemented for a single dependency. Related approaches apply to
dynamically typed languages - Angular.js, for instance, [has a dependency injection framework][angdi].

The same is true of system components - system components should not hard code
uris to remote APIs, and should consider ways to abstract themselves from the
concrete location and implementation of external services. Considerations like
supporting forward proxies between the component under test and the services it
accesses, ensuring that switching between remote service instances is quick and
painless, and minimising the complexity and volume of data on which any single
component depends all make faking a more viable approach. Just as faking an Java
or C# interface with 20 methods is usually harder than faking an interface with
only a couple, if a remote service provides lots of functionality it is likely
to be more difficult to fake it.

## In-Process or Out-Of-Process Fakes?

As you might guess from the above, one of the important distinctions between different faking approaches is whether it depends on any form of IPC actually taking place. I'm using the [general definition of IPC][ipc] here, which includes not only communication over sockets and named pipes, but also via the file system, STDIO pipes and memory mapped files.

In-process fakes offer the opportunity to completely prevent all IPC, thereby allowing tests to be fast and reliable, isolated from packet loss, network latency, disconnection and the other things that make IPC challenging. In this way, they provide a very strong form of isolation and, if reliability is your main concern, are probably the best choice. In some cases, an unreliable or long-running service may depend on no IPC whatsoever, making in-process faking the only option.

However, in-process fakes are not always possible/useful. Code bases must be designed in advance with this faking strategy in mind, and must either create separate builds or separate runtime configurations for production and testing. This isn't the only way that such testing becomes less reliable - further differences between production and test systems are introduced as the actual message placed on the wire isn't checked, and every possible IPC failure scenario must now be programatically simulated. In any highly distributed environment, such a level of abstraction from the concrete form IPC takes is likely to raise concerns - that's exactly where most of the bugs are, after all. And, in some realtime systems, telemetry directly affects validity.
Out-of-process fakes are often easier to insert into an existing system - most system components are likely to be configurable as a matter of necessity, and a certain guaranteed level of abstraction is achieved by the nature of IPC communication (e.g. my Haskell Scotty stub can pretend to be your Ruby Sinatra API because all information is transferred in JSON over HTTP rather than by taking on in-process dependencies). So it's often relatively easy to slot in a fake remote service.

The unreliability of IPC can also be mitigated when fakes can be hosted closer to, or on a more relaible/faster connection to, the running application than production services are. Talking to a locally hosted fake on the tester/developer's machine reduces the risk of unpredictable test failures.

## Descriptive vs Behavioural Faking

The other axis we might want to consider is what kind of work a fake is doing.
Lets say we talk to a cryptographically secure randomization service over HTTP in
our app to get random numbers. So we have an interface like

```csharp
public interface IRandomizationService {
  int NewRandom(int max);
}
```

We could implement a fake that was purely static (returns the same value
regardless of the input):

```csharp
class StaticFake : IRandomizationService {
  public int NewRandom(int max) => 3;
}
```

We could implement a fake that returns a dynamic value dependent on the input:

```csharp
class DynamicFake : IRandomizationService {
  public int NewRandom(int max) => max;
}
```

Or we could take the idea of using dynamic values further and simulate some
subset of the expected behaviour of the service, just without the IPC call:

```csharp
class SimulatorFake : IRandomizationService {
  public int NewRandom(int max) => new Random().Next(max);
}
```

What type of fake you'll need to use depends on the kind of testing you want to
get done.

* The static fake is easy to understand, highly predictable, and will
make your unit tests super fast. It will also restrict your test surface, or
you'll need to create a new class for each return value you want to test.
* The dynamic fake allows you to generate a range of different values, perhaps
based on the input. However, you may need to decide on all the interesting test
cases in advance, based on specific data values if you are not simulating.
* The simulator fake is less predictable. Depending on the current tick count,
the same input will produce different outputs on each run. Instead of having
to know every interesting input and output in advance, tests which use simulators
may specify more abstract rules (rather than "an input of 2 yields a result of
1", you assert for something like "an input in the range 1-4 always yields a
result in the range 5-10", and generate as many scenarios as you can to verify
the rule - a bit like a [QuickCheck][qc] property). In this case,
randomization introduces an extra bit of complexity, but it's common to find cases
in which the number of possible inputs far exceeds the resources we have to devote
to testing them. By simulating the behaviour more closely, we trade predictability
and simplicity for a chance to catch more edge-case bugs at least some of the time.
Simulators may be required when a component has a genuine dependency on the
behaviour, rather than just the data/state, of another component.

These principals apply equally to out-of-process fakes - they can return static
data, dynamically generated data, or simulate the behaviour of the faked service
in some way. Take a fake REST API - you might implement this by mapping static
response files to certain urls in your fake, by dynamically picking a fixed
response, or by actually transitioning the state of a simulation.

## Stubbing vs Mocking vs Service Virtualization

So, at one level, it's all just faking. At another, software vendors have started to use terms like "service virtualization" to distinguish their offerings from those of others. Confusingly, other vendors make no such distinction in terminology. Take WireMock and HoverFly, two very simillar solutions to out-of-process faking. Both allow static, dynamic and simulated faking, yet WireMock is a "mock" server (clue's in the name), whilst SpectoLabs seem to place weight on the distinction between mocking (meaning in-process faking) and service virtualization (meaning out-of-process faking).

A variety of attempts have been made to make the distinctions between stubs, mocks and virtual services clear and fixed. I'm not going to try to establish a general convention on how these terms should be used, but they are convenient monikers so allow me to define what they mean in this discussion:

### Stub:
A stub is a minimal, ideally static, implementation of a dependency that
facilites low-level component testing and debugging. It does not attempt to
simulate behaviour except where not doing so would prevent **any** testing from
taking place.

### Mocks:
Mocks really pretend to be the thing they fake. Mocks may implement as
much behaviour as is required to get whatever test is at hand done. They don't
have to be complete simulations, but they may be complex components in their
own right.

### Virtual Service:
An out-of-process stub or mock.

## WireMock vs HoverFly vs Canned

Let's compare three different approaches to out-of-process faking for HTTP
dependencies. Say we have a client application that communicates with a REST API.

We could

* A: Create a full webserver that stands in for the REST API and returns
canned responses. That's exactly what [Canned][can] (and Dyson and some other tools) do.
You point the client at Canned by setting the hostname and port of the faked server
to the location of the Canned server. When the client makes a request for a resource,
Canned looks up a static file mapped to that resource path, and returns that to
client. Tests can set up the desired server responses for their scenarios by
writing response files to Canned's file system before causing the SUT to
make the request. Canned does not support proxying requests through to other
services, meaning it is less flexible at other stages of testing (e.g. in
integration testing, some urls but not others may need to be stubbed).

* B: Next step up would be to take our mock server and have it act as a reverse
proxy. This is the approach [WireMock][wm] takes - to the client, it looks exactly
like the real dependency, but it has the ability to selectively forward
requests on to the actual service/a more complex mock service, as well as serving
up canned responses. WireMock is also programmable at run time, giving tests more
control over how the mock behaves.

* C: We could take our fake webserver and have it pretend to be network infrastructure. Instead of an opaque reverse proxy pretending to be the real dependency, it could could instead masquerade as a transparent forward proxy, but sometimes return responses it makes up, without forwarding the request at all. This is what [HoverFly][hvf] does (EDIT: HoverFly can also act as a reverse proxy - see Feedback). The difference here is largely one of ease of configuration. For clients which are themselves system components running in a server farm, a forward proxy is often an easy thing to set up. UI clients generally stuggle more to setup proxy configurations. Compare for example the relative ease of switching an iOS or Android configuration file to have the client point at a different url, with the significantly more involved process of getting every simulator and device used to adopt the correct forward proxy settings. This also applies to some extent to browser based clients, which often use the proxy settings of the host. In a case where tests are parrelized on the same machine, tests which program the mock on the fly are highly likely to conflict with each other, since they must all use the same proxy. Automating setting the system proxy on a device inherently has more depencies on the specific OS of the host. This is not a problem faced by reverse proxies however

And that's kind of it really. There are a bunch of vendor related reasons to perhaps perfer HoverFly to WireMock - if you want to use their SPECTO solution as well, for instance. HoverFly is more language agnostic in its middleware, and if no one in your shop is at all familiar with the JVM, the fact that middleware can be written in, say, Python, Node.js, or Erlang will appeal. On the other hand, WireMock offers [stateful behaviour][wmstate] that HoverFly lacks.

However, for me the key difference between Canned and the other two is the lack of any proxying capabilities. It's a super simple, straight forward bit of javascript that works very well for local testing. However, it scales poorly into other kinds of testing, particularly integration environments, because it lacks the ability to act as a proxy as well as a standard webserver.

Choosing between HoverFly and WireMock is then about what kind of proxying best fits your testing. Is it easier to configure a proxy, or to change a host name entry in a config file/DNS table? Given that I'm now testing mobile clients, I think the choice has to be WireMock, or some other capable reverse-proxy. If you are working primarily with backend, containerized services, HoverFly would probably be my choice given the ability to mock multiple remote services in a single HoverFly instance (enabled by the fact that it is a forward proxy). Few of us don't have to consider UI clients, however, and WireMock offers consistency for backend and front end testing.

## Feedback:

Thanks to the awesome chaps at HoverFly for their [detailed response][resp] to this discussion. HoverFly may also act as a [reverse proxy][hfrprox], removing the key configuration differences with WireMock.

[ipc]: https://en.wikipedia.org/wiki/Inter-process_communication#Approaches
[wm]: http://wiremock.org/
[can]: https://github.com/sideshowcoder/canned
[hvf]: http://hoverfly.io/
[qc]: https://www.schoolofhaskell.com/user/pbv/an-introduction-to-quickcheck-testing
[rhino]: https://hibernatingrhinos.com/oss/rhino-mocks
[ninject]: http://www.ninject.org/
[castlew]: http://www.castleproject.org/
[autofac]: https://autofac.org/
[angdi]: https://docs.angularjs.org/guide/di
[resp]: https://github.com/SpectoLabs/hoverfly/issues/417
[wmstate]: http://wiremock.org/docs/stateful-behaviour/
[hfrprox]: https://docs.hoverfly.io/en/latest/pages/keyconcepts/webserver.html
