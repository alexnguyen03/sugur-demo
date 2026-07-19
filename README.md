# Sugur

> Sugar for your cup of Java — build lightweight desktop apps with Java and web technologies.

Sugur is a desktop app framework that pairs a Java backend with a React (or any web) frontend rendered in the OS-native webview, and ships as a single ~20–30MB binary via GraalVM native-image.

## Status

🚧 **Early planning stage.** No code yet — see [plan.md](plan.md) for the full roadmap. Not usable as a library today.

## Why

- **Small binaries.** Native-image + system webview keep the final app compact.
- **Familiar stack.** Java backend, React (or your framework of choice) frontend.
- **Native feel.** Uses the OS's built-in webview (WebView2 / WKWebView / WebKitGTK).

## Planned usage (target experience)

```java
public class Main {
    public static void main(String[] args) {
        Application.builder()
            .window(w -> w.title("Hello Sugur").size(800, 600))
            .run();
    }
}
```

```ts
import { invoke } from "@sugur/runtime";

const user = await invoke("getUser", { id: 1 });
```

## Roadmap

See [plan.md](plan.md) for the detailed, phased roadmap (foundations → PoC app → library extraction → developer experience → v0.1 launch → v1.0).

## License

[MIT](LICENSE)
