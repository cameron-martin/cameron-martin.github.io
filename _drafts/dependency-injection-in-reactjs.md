I have heard people say that JavaScript does not need Java-y concepts like IOC containers, since a functional style is best used. However, this is completely incorrect, as is evidenced by functional languages need for parametric modules.

There are only a [few][di1] articles on how to use DI with React, but none of them seem satisfactory
 They either use a service locator or ???.

The reason why doing dependency injection correctly in react is non-obvious is because you do not instantiate the react classes (the framework does) so you cannot inject dependencies into the constructor. However, classes in JavaScript are first class, 

[di1]: https://www.npmjs.com/package/react-di
