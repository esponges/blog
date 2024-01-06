# React (web) vs React Native


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
