```js
import React from 'react';

class Component extends React.Component {
  static displayName = 'Component';

  static propTypes = {
    requiredProp: React.PropTypes.bool.isRequired
  };

  render() {
    return <div className="foo">hai</div>;
  }
};

export default Component;
```
