# React (web) vs React Native

Both React and React Native are based on the same principles and share the same concepts like components, props, state, etc. However, there are some differences between them.

## State Management

Both React and React Native can use the same state management libraries like `redux`, `apollo-react`, and even `react-query` (now `tanstack-query`). _The implementation is almost the same_ for both platforms.

This is great because the complexity of state management in production apps is usually __high__ and we can basically copy and paste the code from one platform to another.

## Component Libraries and UI Elements

- React (Web): Uses HTML DOM elements (div, span, etc.). Supports styling directly in the components using tailwindcss.
- React Native (Mobile): Uses native components (View, Text, etc.). Styling is done via a JavaScript-based styling system that mimics CSS but with some differences.

e.g.:

```jsx
// React (Web)
<div className="text-red-500">Hello World</div>

// or using css modules
import styles from './styles.module.css'

<div className={styles.text}>Hello World</div>

// React Native (Mobile)
<View style={tailwind('text-red-500')}>Hello World</View>

// or using StyleSheet
const styles = StyleSheet.create({
  text: {
    color: 'red'
  }
})

<View style={styles.text}>Hello World</View>
```

### Considerations

1. There are several solutions to use tailwind with React Native like `tailwind-rn`, `nativewind`, `react-native-tailwindcss`, etc. but they are not as mature as the web version. Where `nativewind` is the most mature one but it is not as feature-rich as the web version.
2. Css modules and StyleSheet share the same concept of scoped styles and could be used to create styles and they are native to the platform. No extra dependencies are required.

## Persisting Data

- React (Web): Uses `localStorage` to persist non sensitive data. To store JWT tokens we use cookies.
- React Native (Mobile): Uses `AsyncStorage` to persist data. This is considered an equivalent to the storage APIs on the web.

Unfortunately the storage APIs on the web and mobile are not compatible with each other and implementations are different.

### Considerations

1. `AsyncStorage` is deprecated and will be removed in future releases. However there are great reliable alternatives such as `@react-native-async-storage/async-storage`.
2. There are solutions for secure storage like `react-native-keychain`, however we should do further research to find the best solution.

### APIs

Web APIs and Native APIs are fundamentally different, primarily because they are designed for different platforms and have different capabilities and limitations. Therefore expect a completely different implementation for each platform. 

## Navigation
Navigation differs a lot between platforms are similar in some ways. For example, both platforms use a stack navigation to navigate between screens. However, the implementation is different. And if you are using nextjs, the implementation is completely different, since nextjs uses a file-based routing system and React Native uses a component-based routing system.

e.g.:

```jsx
// Native
import { createStackNavigator } from '@react-navigation/stack'

const Stack = createStackNavigator()

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Profile" component={ProfileScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  )
}
```

```jsx
// Web using React Router
import { BrowserRouter as Router, Switch, Route } from 'react-router-dom'

function App() {
  return (
    <Router>
      <Switch>
        <Route path="/profile">
          <ProfileScreen />
        </Route>
        <Route path="/">
          <HomeScreen />
        </Route>
      </Switch>
    </Router>
  )
}
```

Using nextjs which is a file-based routing system:

- pages
  - index.js
  - profile.js
  - [id].js

## Props

React Native and React share the same fundamental principles, including the use of props (properties) to pass data and event handlers between components. However, there are differences in the specific props available and their usage, primarily because React Native and React cater to different environments (mobile vs. web).

For example, React has the `onClick` prop for handling click events, while React Native has the `onPress` prop for handling press events. Likewise, React has the `className` prop for styling components, while React Native has the `style` prop for styling components. Many of the props are similar, but they are not the same. There are some web props that are native to the web and the DOM, and there are some native props that are native to the platform.

## Tools for sharing code between platforms

At the moment there are no tools to migrate from one platform to another. However, there are some tools that can help us to share code between platforms like:

- [react-native-web](https://www.npmjs.com/package/react-native-web) which allows rendering React Native components in the browser.
- [nx](https://nx.dev/) which allows us to share code between platforms and even share components between platforms using a [monorepo](https://docs.expo.dev/guides/monorepos/) approach.

From the above tools, I would only recommend using `nx` with very robust automation testing suites. Otherwise, I would recommend to keep the code separated and use the same state management libraries to share the business logic. This is because it would be easy to break the code in one platform and not notice it until we test it in the other platform.

I've created this mono repo to test shared code between a nextjs and an expo app:
https://github.com/esponges/react-expo-shared

## Expo

Expo it's a great alternative to build React Native apps without having to deal with the native side of things: the native side means handling the native code for each platform (iOS and Android). For example, we use the `expo-camera` package for using the camera without needing to have two different implementations per platform. This makes the development process faster and easier, even for web developers. It makes local development and testing easier, and it makes it possible to develop for iOS without a Mac.  For deployment there's EAS (Expo Application Services) —similar to Vercel— which is a set of tools that allow us to build and publish our app to the app stores without having to deal with the native side of things.

While expo is very useful, it has some limitations like: 

- Some libraries are not compatible with expo.
- It is not possible to use native code (Java, Kotlin, Swift, Objective-C) with expo.

Good thing is that you ever _really_ need to use native code, you can always eject (port to Native CLI) from expo and use the native code.

Some great videos about expo:
- https://www.youtube.com/watch?v=q-sKCsscIsc&ab_channel=SimonGrimm
- https://www.youtube.com/watch?v=04Sa3TBOYnw&ab_channel=TomBaranowicz

### Conclusion

Expo has evolved a lot during the last years and it is a great tool to build React Native apps. However, I would only recommend using for small to medium/small size teams. Also you have to be sure that the features you need are compatible with expo. Good thing is that in a worst case scenario, you can always eject from expo and use the native code.

## Wrapping up

React Native is a great tool to build mobile apps. It is based on the same principles as React and it is easy to learn. However, there are some differences between React and React Native that we need to be aware of. A lot of important concepts are the same, but the implementation is different. There's no shortcut for migrating from one platform to another, but there are some tools that can help us to share code between platforms. However, with tools like _Expo_ web developers can build mobile apps without having to deal with the native side of things. This makes the development process faster and easier, even for web developers.
