# phoenix-react

The package offers state support for [Phoenix Channels](https://hexdocs.pm/phoenix/channels.html) using the [React's Context API](https://reactjs.org/docs/context.html). The only external dependency is [Phoenix js](https://hexdocs.pm/phoenix/js/).

It provides :
- Central state for your events
- Simple channel and events registration
- HOC for easy coupling with the Provider

## Getting started

### Install the dependency

Using npm
```sh
npm install --save phoenix-react
```

Using yarn
```sh
yarn add phoenix-react
```

### Setting up the context Provider

```js
import React from 'react'
import ChannelProvider from 'phoenix-react'
import channels from './channels'

//  Refers the initial state you want
const state = {
  messages: [],
}

//  Refering to the `opts` parameter for Pheonix.Socket instanciation
const params = {
  params: {
    token: localStorage.token,
  },
}

export default () => (
  <ChannelProvider
    url="ws://localhost:4000/socket"
    channels={channels}
    state={state}
    params={params}
  >
    <App />
  </ChannelProvider>
)
```

#### Writings channels handler

This `channels` prop of the provider is an array of channel you create with the `createChannel` helper. The first parameter is the topic to subscribe and the second one is a callback that will setup the socket.

Within the callback function, you receive the `channel` as first paramter and the provider value which has `mutate`, `fire`, `state` and `getters`props. I highly recommend creating mutats

```js
import { createChannel } from 'phoenix-react'

//  Messages mutators. These are helpers that will mutate the state withing a payload. Even though these are not required, they are recommended.
const messageMutators = {
  add: (state, message) => ({
    ...state,
    messages: [
      ...state.messages,
      message
    ],
  }),
  set: (state, messages) => ({
    ...state,
    messages: messages,
  }),
}

//  The message channel, the mutate function performs a state mutation of the provider which will re-render the consumer components.
const messageChannel = createChannel('room', (channel, { mutate }) => {

  //  You can add handlers for different events. This one with add a message to the state when the `new_msg` event is broadcasted by the Phoenix server
  channel.on('new_msg', ({ body }) => {
    mutate(state => messageMutators.add(state, body.message))
  })

  //  Don't forget to join the channel. You can also chain `receive(message, body)` to it to handle different server return code. See Phoenix.js documentation for more information
  channel.join()
    .receive('ok', ({ body }) => {
      mutate(state => messageMutators.set(state, body.messages))
    })

  //  Return that channels to it can be stored into the provider
  return channel
})

export default [
  messageChannel,
]
```

### Coupling components to the provider

To bind state mutations to components, there is two way. Wither use the Consumer or the HOC. In either way, you will receive the following payload:

- `fire(channel, event, body)`: function that fires an event to the server on the desired channel.
- `state`: The current state of your provider which **should not** be mutated. Use the mutate function instead.
- `getters`: The getter object you passed to the provider. These are "quick access functions" to your state.
- `mutate`: function that directly mutates the state.
- `leave`: function that leaves the given channel.

#### Using the Consumer

Simply wrap the location where you want to use the provider state by the consumer.

```js
import React from 'react'
import ChannelsProvider from 'phoenix-react'

export default () => (
  <div className="app">
    <div className="message-list">
      <ul>
        <ChannelsProvider.Consumer>
          {({ state }) => state.messages.map(message => (
            <li>[{message.author}]: {message.text}</li>
          ))}
        </ChannelsProvider.Consumer>
      </ul>
    </div>
  </div>
)
```

#### Using the High-order component (HOC)

Just wrap the Component while edefault exporting it. The advantage of the HOC is that you can pass a `mapState` function as second parameter to map the only thing want from the payload.

```js
import React from 'react'
import { withChannels } from 'phoenix-react'

const MessageList = ({ messages }) => (
  <ul>
    {messages.map(message => (
      <li>[{message.author}]: {message.text}</li>
    ))}
  </ul>
)

const WrappedMessageList = withChannels(MessageList, ({ state }) => ({
  messages: state.messages
}))

export default () => (
  <div className="app">
    <div className="message-list">
      <WrappedMessageList />
    </div>
  </div>
)
```

## API

### `ChannelsProvider`



