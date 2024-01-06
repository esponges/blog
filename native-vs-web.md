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

## Tools for sharing code between platforms

At the moment there are no tools to migrate from one platform to another. However, there are some tools that can help us to share code between platforms like:

- [react-native-web](https://www.npmjs.com/package/react-native-web) which allows rendering React Native components in the browser.
- [nx](https://nx.dev/) which allows us to share code between platforms and even share components between platforms using a [monorepo](https://docs.expo.dev/guides/monorepos/) approach.

From the above tools, I would only recommend using `nx` with very robust automation testing suites. Otherwise, I would recommend to keep the code separated and use the same state management libraries to share the business logic. This is because it would be easy to break the code in one platform and not notice it until we test it in the other platform.
