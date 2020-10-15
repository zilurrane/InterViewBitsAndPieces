This is how I understand :

You would agree that react makes thing simple and faster using components . With JSX we can make things easier for user-defined components . End of the day all of it gets translated to pure JavaScript (I assume you understand how React.createElement works)with function calls holding other function calls as its arguments/properties holding yet other function calls and so on .. Anyway nothing for us to worry about as react does this on its own internally .

But how does this gives us an UI ? Why it is faster from other UI libraries ?

<-- ALL HAIL ReactDOM library and the render method -->

An ordinary ReactDOM call looks like this :

// I have avoided the usage of JSX as its get transpiled anyway
ReactDOM.render(
React.createElement(App, { //if any props to pass or child }), // "creating" a component
document.getElementById('#root') // inserting it on a page
);
Heard about VirtualDOM ? { yes : 'Good'} : { no : 'still Good'} ;
The React.createElement construct element object with type and props based on the components we have written and place the child elements under a children key inside props. It recursively does this and populates a final object which is ready to be converted to HTML equivalent and painted to the Browser.

This is what VirtualDOM is, which resides in reacts memory and react performs all its operation on this rather on actual Browser DOM . It looks something like this:

{
type: 'div',// could be other html'span' or user-diff 'MyComponent'
props: {
className: 'cn',
//other props ...
children: [
'Content 1!', // could be a component itself
'Content 2!', // could be a component itself
'Content n!', // could be a component itself
]
}
}
After a Virtual DOM object is built, ReactDOM.render will transform it into a DOM node our browser can pain on to UI according to those rules:

If a type attribute holds a string with a tag name—create a tag with all attributes listed under props. If we have a function or a class under type—call it and repeat the process recursively on a result. If there are any children under props—repeat the process for each child one by one and place results inside the parent’s DOM node.

The Browser paints it to the UI , this is an expensive task . React is very smart to understand this. Updating the component means creation of a new object and paint to UI. Even if a small change is involved it will make the whole DOM tree recreated . So how do we make the Browser never have to create DOM each time rather paint only the necessary things.

This is where we need Reconciliation and the diffing algorithm of React .. Thanks to react we don't have to do it our self manually , its taken care of internally here is a nice article to understand deeper

Now you can even refer the official React docs for Reconsiliation

Few points worth noting :

React implements a heuristic O(n) algorithm based on two assumptions: 1) Two elements of different types will produce different trees. 2) The developer can hint at which child elements may be stable across different renders with a key prop.

In practice, these assumptions are valid for almost all practical use cases. If these are not met it will cause performance issues.

I am just copy Pasting few other points just to give a idea how its done :

Diffing : When diffing two trees, React first compares the two root elements. The behavior is different depending on the types of the root elements.

Scenario 1: type is a string, type stayed the same across calls, props did not change either.

// before update
{ type: 'div', props: { className: 'cn' , title : 'stuff'} }

// after update
{ type: 'div', props: { className: 'cn' , title : 'stuff'} }
That is the simplest case: DOM stays the same.

Scenario 2: type is still the same string, props are different.

// before update:
{ type: 'div', props: { className: 'cn' } }

// after update:
{ type: 'div', props: { className: 'cnn' } }
As type still represents an HTML element,React looks at the attributes of both, React knows how to change its properties through standard DOM API calls, without removing the underlying DOM node from a DOM tree.

React also knows to update only the properties that changed. For example:

<div style={{color: 'red', fontWeight: 'bold'}} />

<div style={{color: 'green', fontWeight: 'bold'}} />
When converting between these two elements, React knows to only modify the color style, not the fontWeight.

///////When a component updates, the instance stays the same, so that state is maintained across renders. React updates the props of the underlying component instance to match the new element, and calls componentWillReceiveProps() and componentWillUpdate() on the underlying instance. Next, the render() method is called and the diff algorithm recurses on the previous result and the new result. After handling the DOM node, React then recurses on the children.

Scenario 3: type has changed to a different String, or from String to a component.

// before update:
{ type: 'div', props: { className: 'cn' } }

// after update:
{ type: 'span', props: { className: 'cn' } }
As React now sees that the type is different, it would not even try to update our node: old element will be removed (unmounted) together with all its children.

It is important to remember that React uses === (triple equals) to compare type values, so they have to be the same instances of the same class or the same function.

Scenario 4: type is a component.

// before update:
{ type: Table, props: { rows: rows } }

// after update:
{ type: Table, props: { rows: rows } }
“But nothing had changed!”, you might say, and you will be wrong.

If type is a reference to a function or a class (that is, your regular React component), and we started tree reconciliation process, then React will always try to look inside the component to make sure that the values returned on render did not change (sort of a precaution against side-effects). Rinse and repeat for each component down the tree—yes, with complicated renders that might become expensive too!

To make sure such things come clean:

class App extends React.Component {

state = {
change: true
}

handleChange = (event) => {
this.setState({change: !this.state.change})
}

render() {
const { change } = this.state
return(
<div>
<div>
<button onClick={this.handleChange}>Change</button>
</div>
{
change ?
<div>
This is div cause it's true
<h2>This is a h2 element in the div</h2>
</div> :
<p>
This is a p element cause it's false
<br />
<span>This is another paragraph in the false paragraph</span>
</p>
}
</div>
)
}
}
Children =============================>

we also need to account for React’s behavior when an element has more than one child. Let’s say we have such an element:

// ...
props: {
children: [
{ type: 'div' },
{ type: 'span' },
{ type: 'br' }
]
},
// ...
And we want to shuffle those children around:

// ...
props: {
children: [
{ type: 'span' },
{ type: 'div' },
{ type: 'br' }
]
},
// ...
What happens then?

If, while “diffing”, React sees any array inside props.children, it starts comparing elements in it with the ones in the array it saw before by looking at them in order: index 0 will be compared to index 0, index 1 to index 1, etc. For each pair, React will apply the set of rules described above.

React has a built-in way to solve this problem. If an element has a key property, elements will be compared by a value of a key, not by index. As long as keys are unique, React will move elements around without removing them from DOM tree and then putting them back (a process known in React as mounting/unmounting).

So Keys should be stable, predictable, and unique. Unstable keys (like those produced by Math.random()) will cause many component instances and DOM nodes to be unnecessarily recreated, which can cause performance degradation and lost state in child components.

Because React relies on heuristics, if the assumptions behind them are not met, performance will suffer.

When state changes: =========================================>

Calling this.setState causes a re-render too, but not of the whole page, but only of a component itself and its children. Parents and siblings are spared. That is convenient when we have a large tree, and we want to redraw only a part of it.