I have heard people say that JavaScript does not need Java-y concepts like IOC containers, since a functional style is best used. However, this is completely incorrect, as is evidenced by functional languages need for parametric modules.

There are only a [few][di1] articles on how to use DI with React, but none of them seem satisfactory
 They either use a service locator or ???.

The reason why doing dependency injection correctly in react is non-obvious is because you do not instantiate the react classes (the framework does) so you cannot inject dependencies into the constructor. However, classes in JavaScript are first class, so you can write functions that take as arguments a component's depdendencies and returns a react component. E.g.

```javascript
export default TodoItem => class TodoList extends React.Component {
  render() {
    return (
      <ul>
        <TodoItem id={1} title="Buy milk" />
        <TodoItem id={2} title="Write more blog posts" />
      </ul>
    );
  }
}
```

```javascript
export default todoItemService => class TodoItem extends React.Component {
  _handleDelete() {
    todoItemService.delete(this.props.id);
  }
  
  render() {
    return (
      <li>
        {this.props.title}
        <button onClick={this._handleDelete} />
      </li>
    );
  }
}

So, we now have dependency injection that is succinct, and most importantly and unlike the service locator based solutions, the dependencies are explicit.

[di1]: https://www.npmjs.com/package/react-di
