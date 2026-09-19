# @assistant-ui/react — React Native Compatibility

## Why @assistant-ui/react?

`@assistant-ui/react` is the recommended chat UI library for OpenAI SDK applications. It eliminates hours of boilerplate by handling the most complex parts of AI chat out of the box: streaming responses, tool/function calling visualization, message history management, and platform-aware rendering. Building a production-quality chat UI from scratch requires managing streaming state, loading indicators, error boundaries, markdown rendering, and mobile touch interactions — this package does all of that with one component.

## Benefits

- **Single codebase for web + mobile** — renders natively on browsers (React), React Native (Expo), and terminals
- **Streaming built-in** — real-time token-by-token responses without custom WebSocket logic
- **Tool calling visualization** — automatically shows function calls, results, and error states in the chat stream
- **Zero boilerplate integration** — works with OpenAI SDK out of the box via `@assistant-ui/react-openai` adapter
- **Markdown rendering** — built-in syntax highlighting, code blocks, and rich text formatting
- **Extensible architecture** — customize any component (bubbles, inputs, loading states) while keeping the core logic intact

## React Native Support

`@assistant-ui/react` supports **React Native** out of the box, not just web.

- Docs: https://www.assistant-ui.com/docs/react-native
- This means a single chat UI codebase works on both **web** and **mobile (iOS/Android)** via Expo.
