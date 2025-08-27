# 第 0 部分：AI 操作协议

本节定义了 AI 审查员必须遵守的操作协议。在处理本文档中的任何其他规则之前，必须遵守此协议。

### **基本沟通协议**

- **语言要求**：用英语思考，但始终以中文表达最终输出。
- **表达风格**：直接、尖锐、切中要害。如果代码很糟糕，要准确解释原因。
- **技术第一**：批评应始终针对技术问题，而不是个人。不要为了“友好”而软化技术判断。

### **核心指令：先观察，后断言**

该指令旨在防止 AI “幻觉”并确保所有发现都基于事实证据。

1.  **禁止假设**：AI 审查员**绝不能**根据文件名、位置或常见设计模式来假设或预测文件内容。所有分析都必须基于实际的文件内容。

2.  **强制读取**：在应用任何需要代码分析的规则（特别是那些 `engine: "contextual"` 或 `engine: "manual"` 的规则）之前，AI 审查员**必须**首先发出 `read_file` 命令来检索目标文件的确切和完整内容。

3.  **引用证据**：所有报告的违规行为**必须**由强制读取步骤中检索到的实际代码直接支持。AI 必须准备好引用构成违规的具体代码行。

### **技术栈与框架**

本规范用于指导 React、Redux Toolkit、Taro 开发框架下的开发。在开始审查前，请先确认当前工程使用的技术栈，不要给出超出工程技术栈的建议、结论。

- 查看官方文档
  - resolve-library-id - 解析库名到 Context7 ID
  - get-library-docs - 获取最新官方文档

---

# 第 1 部分：架构原则

### 1. 4个思考维度

- **第 1 层：数据结构分析**

  - “糟糕的程序员担心代码。优秀的程序员担心数据结构。”
  - 核心数据是什么？它们之间有何关联？
  - 数据流向何方？谁拥有它？谁修改它？
  - 是否存在不必要的数据复制或转换？

- **第 2 层：特殊情况识别**

  - “好的代码没有特殊情况。” - *《代码整洁之道》*
  - 识别所有 `if/else` 分支。
  - 哪些是真正的业务逻辑，哪些是为糟糕设计打的补丁？
  - 能否重新设计数据结构以消除这些分支？

- **第 3 层：复杂性审查**

  - “如果一个实现需要超过 3 层的缩进，请重新设计它。”
  - 这个功能的本质是什么？（用一句话解释）
  - 当前的解决方案用了多少个概念来解决它？
  - 能否减少一半？再减少一半？


- **第 4 层：可读性优先**
  - “代码是写给人看的，不是写给机器看的”

### 2. 逻辑分层：流程 vs. 规则

> **规则**：对于复杂的业务逻辑，应将其分为**流程 (Process)** 与 **规则 (Rule)**。无论是否分层，都必须遵循单向依赖：流程可以调用规则，但规则绝不能调用流程。
>
> ```yaml
> id: ARCH-003
> description: '纯函数（规则）不能调用流程（Hook 或 Thunk）。'
> severity: 'error'
> engine: 'contextual'
> config:
>   guideline: |
>     当逻辑变得复杂时，为了保持清晰，应将其拆分为“流程”和“规则”两个角色。无论逻辑是否拆分，它们都必须遵循单向依赖的硬性原则。
>
>     1.  **定义角色**:
>         -   **规则 (Rule)**: 负责独立业务计算或验证的**纯函数**。它们无副作用，不依赖于 React Hooks。例如: `calculateTotalPrice()`。
>         -   **流程 (Process)**: 负责编排业务场景的**自定义 Hook** 或 **Thunk**。它们管理状态和副作用。例如: `useFetch()`。
>
>     2.  **审查依赖关系 (硬性规定)**:
>         -   **允许的调用**:
>             -   UI → **流程**
>             -   **流程** → **规则**
>             -   **流程** → **流程**
>             -   **规则** → **规则**
>         -   **禁止的调用**:
>             -   **规则** → **流程**
>
>     3.  **标记违规**:
>         -   任何一个纯函数（规则）内部，如果包含了对自定义 Hook 或 Thunk（流程）的调用，则必须标记为违规。
>
>     **示例：**
>     ```javascript
>     // 规则 (纯函数)
>     const calculateDiscount = (price, rate) => price * rate;
>
>     // 流程 (Hook)
>     function useDiscount(price) {
>       const { discountRate } = useAuth(); // 另一个流程
>       // ✅ 正确: 流程调用规则
>       return calculateDiscount(price, discountRate);
>     }
>
>     // ❌ 错误: 规则不能调用流程
>     function calculateFinalPrice(price) {
>       // 严重违规！纯函数内部不能使用 Hook
>       const { taxRate } = useTaxContext(); // ❌
>       return price * (1 + taxRate);
>     }
>     ```
> ```

### 3. 模块设计与封装

> **规则**：代码应遵守粒度限制以保持可读性。
>
> ```yaml
> id: STYLE-001
> description: '强制函数和文件的行数限制。'
> severity: 'warning'
> engine: 'linter'
> config:
>   file_lines: { warning: 250, error: 400 }
>   function_lines: { warning: 100, error: 200 }
> ```

> **规则**：公共成员必须放在文件的顶部，在私有成员之前。
>
> ```yaml
> id: STYLE-002
> description: '公共（导出的）成员必须在私有（未导出的）成员之前定义。'
> severity: 'warning'
> engine: 'contextual'
> config:
>   guideline: |
>     1. 解析文件以识别所有顶级声明。
>     2. 找到*最后一个* `export` 语句的行号。
>     3. 找到*第一个*未导出的顶级 `function` 或 `const` 声明的行号。
>     4. 如果第一个私有成员出现在最后一个公共成员之前，则标记为违规。
> ```

> **规则**：`index` 文件必须有单一、明确的角色，由其扩展名决定。禁止角色混合。
>
> ```yaml
> id: ARCH-002
> description: '根据扩展名对 index 文件强制执行严格的关注点分离。'
> severity: 'error'
> engine: 'contextual'
> config:
>   guideline: |
>     在任何给定的组件目录中（例如 `src/components/my-component/`），`index` 文件的角色由其扩展名严格定义，以强制实现明确的关注点分离。
>
>     **1. 组件实现文件 (`index.tsx` 或 `index.jsx`)**
>     - **角色**：此文件的唯一目的是实现和导出组件本身。
>     - **必须**：包含 React 组件的实现（JSX、逻辑、`export default MyComponent`）。
>     - **禁止**：包含任何重新导出的语句（例如 `export * from './types'`）。
>
>     **2. 模块桶文件 (`index.ts` 或 `index.js`)**
>     - **角色**：此文件的唯一目的是通过从其他文件重新导出成员来定义模块的公共 API。
>     - **必须**：只包含重新导出的语句（例如 `export * from './types'`、`export { useMyHook } from './hooks'`）。
>     - **禁止**：包含任何组件实现、Hook、类或其他具体逻辑。
>
>     **3. 互斥**
>     - 一个目录**禁止**同时包含 `index.tsx`（或 `.jsx`）和 `index.ts`（或 `.js`）。模块的入口点必须是明确的。
>   steps: |
>     1. 识别 `index.ts`、`index.js`、`index.tsx` 或 `index.jsx` 文件。
>     2. **强制**：读取已识别文件的全部内容。
>     3. **分析内容以查找违规**：
>         - **违规**：`index.tsx` 包含重新导出的语句。
>         - **违规**：`index.ts` 包含具体逻辑或组件实现，而不是只有重新导出。
>         - **违规**：一个目录同时包含 `index.ts` 和 `index.tsx`。
> ```

### 4. 禁止简单的包装器

> **规则**：抽象必须提供**重大价值**。
> 
> ```yaml
> id: ARCH-004
> description: '避免提供无重大价值的简单包装器。'
> severity: 'warning'
> engine: 'contextual'
> config:
>   guideline: |
>     1. 识别潜在的包装器组件（只渲染一个其他组件，传递大部分 props）。
>     2. 读取包装器和被包装元素的代​​码。
>     3. 评估“价值”：它是否简化了复杂的 API、注入了上下文或添加了重要的逻辑/样式？
>     4. 如果它只是重命名或添加一个简单的 `div`，则将其标记为潜在违规。
> ```

### 5. 自定义 Hook 设计

> **规则**：自定义 Hook 的设计必须有单一、明确的目的：要么提供一个可复用的有状态值，要么封装一个自包含的副作用。它不应该是对单个原生 Hook 的简单包装，也不应该模糊地混合多种关注点。
> 
> ```yaml
> id: ARCH-001
> description: '自定义 Hook 必须有明确、单一的目的，要么提供状态，要么封装副作用。'
> severity: 'warning' # 'warning' 比 'error' 更好，因为这更像是一个架构指南
> engine: 'manual' # 这很难用 AST 强制执行，需要人工审查
> config:
>   guideline: |
>     根据以下标准审查自定义 Hook（以 `use` 开头的函数）。标记任何违反这些原则的 Hook：
>
>      1.  **它是“状态提供者” Hook 吗？**
>          - **目的**：提供一个有状态的值和更新它的方法。
>          - **特征**：它通常调用 `useState`、`useReducer` 或 `useContext`。它**必须**返回有状态的值。
>          - **示例（良好）**：`useToggle`、`useCounter`、`useAuth`（来自 context）。
>          - **违规**：一个名为 `useUserName` 的 Hook，它只返回 `const [name, setName] = useState('')` 而不做其他任何事情。这是一个简单的包装，与直接在组件中使用 `useState` 相比没有增加任何价值。
>
>      2.  **它是“副作用”（无头）Hook 吗？**
>          - **目的**：封装一个具有明确生命周期（设置和清理）的副作用。
>          - **特征**：它通常调用 `useEffect` 或 `useLayoutEffect`。它通常**不**返回值，或者返回一个状态指示器（例如 `{ isLoading }`）。
>          - **示例（良好）**：`useEventListener`、`useFetch`、`useDebounce`、`useBackHandler`。
>          - **违规**：一个名为 `useLogOnMount` 的 Hook，它只包含 `useEffect(() => { console.log('mounted') }, [])`。虽然功能正常，但它通常过于零碎。最好有一个更通用的 Hook，如 `useLogger`。
>
>      3.  **它是否混合了关注点？**
>          - 单个 Hook 不应试图管理多个不相关的状态或副作用。例如，一个既获取数据又管理表单输入状态的 Hook 可能做得太多了。它应该被拆分为 `useFetch` 和 `useForm`。
> 
> ```

> **规则**：一个自定义**流程 Hook** 不能调用另一个自定义**流程 Hook**。
> 
> ```yaml
> id: ARCH-005
> description: '为自定义流程 Hook 强制执行明确的组合策略，优先考虑组件级组合而不是嵌套。'
> severity: 'warning' # 从 'error' 更改，因为它现在是一个指南，而不是硬性规定。
> engine: 'manual'
> config:
>   guideline: |
>     需要来自另一个流程 Hook 的数据或功能的过程 Hook 必须遵循优先级顺序：
>
>     1.  **优先级 1（首选）：组件级组合。** 组件应调用两个 Hook，使依赖关系明确。这是最清晰的模式。
>
>     2.  **优先级 2：提取纯逻辑（规则）。** 如果依赖关系是计算逻辑（而不是状态或副作用），则将其提取到一个纯函数中，两个 Hook 都可以调用该函数。
>
>     3.  **优先级 3（谨慎使用）：直接嵌套。** 只有在上述模式不可行时，才允许在严格条件下将一个流程 Hook 直接嵌套在另一个流程 Hook 中：
>         - 它必须是单层深度（A -> B 可以；A -> B -> C 禁止）。
>         - 它主要用于调用选择器 Hook 或无头 UI Hook。
>         - 嵌套具有副作用的自定义过程 Hook 需要团队审查。
>
>     标记任何违反此优先级的代码（例如，在组件级组合简单明了的情况下使用嵌套）。
> ```

### 6. Thunk 设计

> **规则**：Thunk 的调用链深度一般不应超过 3 层。极端场景需提请人工审查。
>
> ```yaml
> id: ARCH-008
> description: '识别深度超过 3 层的 Thunk 调用链，并提请人工审查其必要性。'
> severity: 'warning' # 这是一个需要人工介入的警告
> engine: 'manual'
> config:
>   guideline: |
>     Thunk 调用栈过深会使逻辑难以追踪。深度超过 3 层是潜在的设计问题，需要人工确认。
>
>     1.  **常规深度限制**:
>         -   `A -> B -> C` (✅ 合规)
>         -   `A -> B -> C -> D` (⚠️ 需要人工审查)
>
>     2.  **警惕参数透传**: 如果一个中间层的 Thunk 只是将参数原封不动地传递给下一个 Thunk，而没有处理任何实质性的逻辑，那么它可能是一个多余的层级，应该被重构。
>
>     3.  **AI 审查员操作指南**:
>         -   **步骤 1: 定位起点**: 使用 `search_file_content` 搜索 `dispatch\(.*Thunk.*\)` 来定位所有顶层 Thunk 的调用位置。
>         -   **步骤 2: 追踪调用链**: 对每个起点，使用 `read_file` 读取 Thunk 文件内容，递归搜索下一个被 `dispatch` 的 Thunk。
>         -   **步骤 3: 识别并报告**: 每深入一层，计数器加一。如果深度超过 3，**不直接标记为违规**，而是生成一条警告，指出文件和函数名，并**要求人工审查**该设计的合理性。
> ```

> **规则**：Thunk 必须用于编排与 Redux store 交互的异步或多步骤流程。
>
> ```yaml
> id: ARCH-009
> description: '为 Thunk 的使用场景提供明确的判断标准，防止滥用。'
> severity: 'warning'
> engine: 'manual'
> config:
>   guideline: |
>     在创建一个 Thunk 之前，必须通过以下“试金石测试”来确认其必要性。如果任一问题的答案为“否”，则应考虑使用其他模式（如自定义 Hook 或普通异步函数）。
>
>     1.  **问题一：它是否需要访问 Redux Store？**
>         -   流程是否需要通过 `getState()` 读取 state，或通过 `dispatch()` 提交 action？如果逻辑与 Redux store 完全无关，它就不应该是 Thunk。
>
>     2.  **问题二：它是否在编排一个多步骤流程？**
>         -   流程是否包含多个串行或并行的步骤，尤其是在异步操作的不同阶段 dispatch 多个 action（例如 `pending`, `fulfilled`, `rejected`）？如果只是为了封装单个 action 或一个简单的异步请求，则可能过度设计了。
>
>     3.  **问题三：它的逻辑是否属于全局业务流程？**
>         -   这个流程是否代表一个核心的、可能被多处复用的业务逻辑（如用户登录、支付）？如果逻辑只服务于单个组件的视图状态或生命周期，应优先使用自定义 Hook。
> ```

### 7. 组件设计

> **规则**：父组件**禁止**实现决定子组件*内部结构*的复杂条件逻辑。
> 
> ```yaml
> id: ARCH-006
> description: '子组件必须根据 props 负责自己的内部呈现。'
> severity: 'warning'
> engine: 'manual'
> config: "审查 JSX。如果父组件包含操作子组件内部结构（例如，显示子组件的标题）的复杂逻辑（`&&`、`? :`），则标记它。父组件决定*渲染什么*，子组件决定*如何渲染*。"
> ```

### 8. 优先使用纯函数处理逻辑

> **规则**：如果逻辑不依赖于 React 状态、上下文或生命周期 Hook，则必须将其实现为纯函数。
> 
> ```yaml
> id: ARCH-007
> description: '如果逻辑不依赖于 React 状态、上下文或生命周期 Hook，则必须将其实现为纯函数。'
> severity: 'warning'
> engine: 'manual'
> config:
>   guideline: |
>     该规则强制执行何时创建自定义 Hook 与纯函数的基本原则。
> 
>     **黄金法则**：自定义 Hook 的唯一目的是“封装和重用有状态逻辑”。根本区别在于**自定义 Hook 可以调用其他 React Hook**（例如 `useState`、`useEffect`），而常规函数不能。
> 
>     **试金石测试**：在创建 `use...` 函数之前，请问自己这个问题：
>     > “我想要封装的逻辑是否需要读取或写入 React 状态，或者使用 React 的生命周期或上下文？”
> 
>     ---
> 
>     #### **如果答案是肯定的，它必须是一个自定义 Hook。**
> 
>     将自定义 Hook 用于：
>     1.  **封装复杂、可重用的有状态逻辑**：当您在多个组件中发现类似的 `useState` 和 `useEffect` 组合时。
>         - **示例**：`useCounter()`、`useToggle(initialValue)`、`useFetch(url)`。
>     2.  **抽象和监听浏览器/设备 API**：包装涉及事件侦听器和清理的本机 API 交互。
>         - **示例**：`useEventListener('click', handler)`、`useLocalStorage('my-key')`。
>     3.  **简化上下文消费**：为上下文提供清晰的 API。
>         - **示例**：`useTheme()`，它在内部调用 `useContext(ThemeContext)`。
> 
>     ---
> 
>     #### **如果答案是否定的，它必须是一个常规的纯函数。**
> 
>     这是一个反模式。不要将自定义 Hook 用于：
>     1.  **纯计算或数据转换**：如果函数只是根据其输入计算结果。
>         - **项目违规示例**：`useUpload` Hook 违反了此规则。
>         - **一般违规示例**：
>           ```javascript
>           // ❌ 错误：滥用 useMemo 创建不必要的 Hook
>           const useFormattedPrice = (price) => useMemo(() => `${price.toFixed(2)}`, [price]);
>
>           // ✅ 正确：这是一个纯转换，应该是一个常规函数
>           const formatPrice = (price) => `${price.toFixed(2)}`;
>           ```
>     2.  **对原生 Hook 的无价值包装**：如果 Hook 与其调用的原生 Hook 相比没有增加任何有意义的逻辑。
>         - **项目违规示例**：`useRouteParams` Hook 违反了此规则。
>         - **一般违规示例**：
>           ```javascript
>           // ❌ 错误：此 Hook 没有增加任何价值。
>           function useInputState() {
>             return useState('');
>           }
>           ```
> 
> ```
> 
> ```

---

# 第 2 部分：前端编码细节

### 文件和目录

> **规则**：所有文件和目录**必须**使用 `kebab-case` 样式。
> 
> ```yaml
> id: FS-001
> description: '对所有文件和目录名强制使用 kebab-case。'
> severity: 'error'
> engine: 'filesystem'
> config:
>   patterns: ['*[A-Z]*', '*_*']
>   exclude:
>     - '.git/'
>     - 'node_modules/'
>     - '**/*.png'
>     - '**/*.jpg'
>     - '**/*.jpeg'
>     - '**/*.gif'
>     - '**/*.svg'
>     - '**/*.webp'
>     - '**/*.ttf'
>     - '**/*.otf'
>     - '**/*.woff'
>     - '**/*.woff2'
> ```
> 
> **规则**：组件文件夹不能包含嵌套的“components”目录。
> 
> ```yaml
> id: FS-002
> description: "组件文件夹不能包含嵌套的“components”目录。"
> severity: 'error'
> engine: 'filesystem'
> config:
>   patterns:
>     - 'src/components/*/components'
>     - 'src/pages/*/components/*/components'
>   guideline: |
>     根据项目结构：
>     - `src/components/{name}` 下的文件夹是“组件”。
>     - `src/pages/{page}/components/{name}` 下的文件夹也是“组件”。
>     组件文件夹不能包含其自己的嵌套 `components` 目录。此规则可防止组件结构过于深入和复杂。
> ```
> 
> **规则**：只包含 TypeScript 类型定义的文件**必须**使用 `.d.ts` 后缀。
> 
> ```yaml
> id: FS-003
> description: '只包含类型定义的文件必须使用 .d.ts 后缀。'
> severity: 'error'
> engine: 'regex'
> config:
>   pattern: "^\s*(export|declare)\s+(type|interface)"
>   include: ['*.ts', '*.tsx']
>   exclude: ['*.d.ts']
>   guideline: "如果文件只包含类型/接口导出，则必须将其重命名为 `.d.ts`。"
> ```

### 命名约定

> **规则**：对变量、函数和类型强制执行严格的命名约定。
> 
> ```yaml
> id: NAMING-001
> description: '对变量、函数、类和常量强制执行命名约定。'
> severity: 'error'
> engine: 'linter'
> config:
>   # 对应于 ESLint 规则 `@typescript-eslint/naming-convention`
>   rules:
>     - { selector: 'variable', format: ['lowerCamelCase', 'UPPER_CASE_SNAKE_CASE'] }
>     - { selector: 'function', format: ['lowerCamelCase'] }
>     - { selector: 'class', format: ['UpperCamelCase'] }
>     - { selector: 'variable', types: ['boolean'], format: ['PascalCase'], prefix: ['is', 'has', 'can', 'should'] }
>     - { selector: 'customHook', format: ['PascalCase'], prefix: ['use'] }
> ```
> 
> **规则**：名称必须准确、一致地描述其行为。
> 
> ```yaml
> id: NAMING-002
> description: '名称必须具有描述性、一致性，并且不能产生误导。'
> severity: 'warning'
> engine: 'manual'
> config: "审查名称的清晰度。'handle(data)' 是否描述了它的作用？术语 'user' 是否一致使用（例如 `getUser`、`updateUser`），而不是与 `profile` 混合使用？名为 `getUserData` 的函数是否会误导性地执行异步操作？"
> ```

### 代码风格和逻辑

> **规则**：函数**禁止**接受回调参数。它**必须**返回值或 `Promise`。
> 
> ```yaml
> id: STYLE-003
> description: '函数禁止使用回调返回结果；它们必须使用返回值或 Promise。'
> severity: 'error'
> engine: 'ast'
> config:
>   query: 'FunctionDeclaration[params.name=/callback|cb|onSuccess|onError|onConfirm|onCancel/]'
> ```
> 
> **规则**：Promise 链必须是扁平的；禁止嵌套 `.then()`。
> 
> ```yaml
> id: STYLE-004
> description: '禁止嵌套或混合风格的 Promise 处理；每个函数使用一致的风格。'
> severity: 'error'
> engine: 'ast'
> config:
>   query: "CallExpression[callee.property.name='then'] > ArrowFunctionExpression > BlockStatement > ExpressionStatement > CallExpression[callee.property.name='then']"
>   guideline: |
>     为确保异步代码的可读性和可维护性，此规则旨在消除混乱的 Promise 处理模式。审查期间必须遵守以下三点：
> 
>     1.  **[禁止] Promise 嵌套**：
>         -   `.then()` 回调函数不应包含另一个 `.then()`。这是一个必须避免的反模式。
>         -   ❌ **错误示例**：
>             ```javascript
>             fetchData().then(result => {
>               // 严重违规：在 .then() 中出现另一个 .then()
>               return processData(result.id).then(processed => { // ❌
>                 console.log(processed);
>               });
>             });
>             ```
> 
>     2.  **[禁止] 混合风格**：
>         -   在 `async` 函数内部，如果已经使用了 `await`，则任何 Promise 都不应再使用 `.then()` 或 `.catch()`。整个函数必须保持单一风格。
>         -   ❌ **错误示例**：
>             ```javascript
>             async function loadUserData(id) {
>               const user = await getUser(id); 
>               // 违规：在同一函数中混合使用 .then()
>               fetchRoles(user.id).then(roles => { // ❌
>                 console.log(roles);
>               });
>             }
>             ```
> 
>     3.  **[允许] 两种扁平化风格**：
>         -   只要不违反上述两条规则，开发人员可以自由选择以下任一风格。两种风格均被视为**合规**。
>         -   ✅ **风格 A：扁平 .then 链**：
>             ```javascript
>             function loadUserData(id) {
>               return getUser(id)
>                 .then(user => fetchRoles(user.id))
>                 .then(roles => console.log(roles))
>                 .catch(err => console.error(err));
>             }
>             ```
>         -   ✅ **风格 B：async/await**：
>             ```javascript
>             async function loadUserData(id) {
>               try {
>                 const user = await getUser(id);
>                 const roles = await fetchRoles(user.id);
>                 console.log(roles);
>               } catch (err) {
>                 console.error(err);
>               }
>             }
>             ```
> ```
> 
> **规则**：**严格禁止**嵌套三元运算符。
> 
> ```yaml
> id: STYLE-005
> description: '禁止嵌套三元运算符。'
> severity: 'error'
> engine: 'ast'
> config:
>   query: "ConditionalExpression[alternate.type='ConditionalExpression']"
> ```
> 
> **规则**：在存在性检查后避免多余的空值检查。
> 
> ```yaml
> id: STYLE-006
> description: '在变量已确认存在后，避免多余的空值检查（例如 `|| {}`）。'
> severity: 'warning'
> engine: 'contextual'
> config:
>   guideline: "在函数作用域内，找到一个 `if (!variable) return;` 检查。在该检查*之后*的代码中，查找该 `variable` 的使用情况，其中包含可选链（`?.`）或默认回退（`|| {}`）。如果找到，则将其标记为多余。"
> ```
> 
> **规则**：`setTimeout`/`setInterval` 必须有注释说明其用途。
> 
> ```yaml
> id: STYLE-007
> description: '计时器（setTimeout、setInterval）之前必须有解释性注释。'
> severity: 'error'
> engine: 'regex'
> config:
>   pattern: "(?<!//.*\n.*)setTimeout|(?<!//.*\n.*)setInterval"
>   include: ['*.ts', '*.tsx']
> ```
> 
> **规则**：不要在没有任何处理的情况下捕获异常。
> 
> ```yaml
> id: STYLE-008
> description: '禁止空的 catch 块。'
> severity: 'error'
> engine: 'linter'
> config:
>   # 这是 `eslint-plugin-react-hooks` 和 Redux Toolkit 的 `immutable-state-invariant` 中间件等库的关键功能。
>   rule: 'no-empty-catch'
> ```

### 状态管理

> **规则**：不要直接改变状态。
> 
> ```yaml
> id: STATE-001
> description: '状态必须被视为不可变的。禁止直接改变。'
> severity: 'error'
> engine: 'linter'
> config:
>   # 这是 `eslint-plugin-react-hooks` 和 Redux Toolkit 的 `immutable-state-invariant` 中间件等库的关键功能。
>   guideline: '确保配置 linting 规则以检测直接的状态改变（例如，在 Redux reducer 中 `state.user = ...`）。'
> ```
> 
> **规则**：不要在状态中存储可以从其他状态或 props 派生或计算的数据。
> 
> ```yaml
> id: STATE-002
> description: '避免在状态中存储派生数据。'
> severity: 'warning'
> engine: 'manual'
> config:
>   guideline: '审查组件状态（`useState`）。如果一个状态变量可以在渲染期间即时计算（例如 `const fullName = `${firstName} ${lastName}``），它就不应该是一个单独的状态变量。标记任何此类实例。'
> ```