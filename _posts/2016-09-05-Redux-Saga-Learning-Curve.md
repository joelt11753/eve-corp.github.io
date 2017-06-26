---
layout: page
title: Getting over the learning curve for Redux-Saga
keywords: node, redux-saga, redux, generator function, es6, selectors, reselect
published: false
---

This article is for people who have heard of Redux-Saga and are trying to integrate it into their application.  It is basically a collection of all the areas where I felt lost and/or struggled. 

For more information, see the [Official Redux-Saga Website](http://yelouafi.github.io/redux-saga) and the [Official Redux-Saga Documentation](http://yelouafi.github.io/redux-saga/docs)


# A Complete Example

** Feel free to skip down to the [confusing bits](#confusing-bits)
``` jsx
// myContainer.js
import * as actions from './actions'
import * as selectors from './selectors'

// export the real class for testability
export class MyContainer extends React.Component{
    render(){
        return(){
            <div>
            <input type="text" value="this.props.inputText" onChange={this.props.inputChanged} />
            <span onClick="{this.props.handleClick}"> Click Me </span>
            </div>
        };
    }
}

const mapStateToProps = selectors.selectMyContainer();

function mapDispatchToProps(dispatch) {
  return {
    //dispatch, // I'm not a fan of including dispatch since it leaks the abstraction redux gives us
    handleClick(e) {
      e.preventDefault();
      dispatch(actions.inputSubmitted());
    },
    inputChanged(e) {
      dispatch(actions.inputChanged(e.target.value));
    },
  };
}

// export default as a connected redux container
export default Connect(mapStateToProps, mapDispatchToProps)(MyContainer);

```

``` js
// actions.js
import {INPUT_SUBMIT, INPUT_CHANGE} from '.\constants';

// all this does is return an action, which gets used by our call to `dispatch` in `mapStateToProps`
function inputSubmitted(value){
    returns {
        type: INPUT_CHANGE,
        payload: value
    }
}

function inputSubmitted(){
    returns {
        type: INPUT_SUBMIT
    }
}

```

``` js
// reducers.js



```

```
// sagas.js


```

## Explanation of the Complete Example

TODO

# The Confusing Bits <a name="confusing-bits"></a>

## How to integrate sagas with my action (creators)?

In short, and in contrast to [redux-thunk](TODO), to use sagas as designed, your `actions` file will end up very sparse.  This is because `redux-saga` has its own mechanism for listening to events.

`actions` no longer take action, they become only `action creators` -- functions to create actions, where an action is an object of the form

``` JavaScript
{
    type: 'MY_ACTION_NAME',
    //...(anything else, `payload` by convention)
}

```


## Does it have to be a generator function?

## What's with `yield` everywhere?

## Why isn't my `takeEvery` or `takeLatest` working?

A) You have the wrong import

These are not in the `redux-saga\effects` module, but rather just in `redux-saga`.   

Ensure your import statement looks like `import { takeEvery, takeLatest } from 'redux-saga';` 

B) You forgot to put the `*`, as in `yield* takeEvery(MY_ACTION_NAME, handleAction)` 

## What's the "best practice" for listening to events?

## How do I listen to something like `socket.io`?