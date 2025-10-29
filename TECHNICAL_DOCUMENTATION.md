# Online Python Tutor - Technical Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Source Code Structure](#source-code-structure)
3. [Data Flow Architecture](#data-flow-architecture)
4. [Technology Stack](#technology-stack)
5. [Modern Stack Comparison](#modern-stack-comparison)
6. [VSCode Plugin Feasibility](#vscode-plugin-feasibility)
7. [Key Components Deep Dive](#key-components-deep-dive)
8. [Development Workflow](#development-workflow)

---

## Project Overview

**Online Python Tutor** (OPT) is an educational tool that helps people learn programming by visualizing code execution step-by-step. It supports multiple programming languages including Python, Java, JavaScript, TypeScript, Ruby, C, and C++.

**Website**: http://pythontutor.com/  
**Author**: Philip Guo (philip@pgbovine.net)  
**License**: BSD-3-Clause

### Core Functionality
- Step-by-step code execution visualization
- Memory state visualization (stack frames and heap objects)
- Variable value tracking
- Support for multiple programming languages
- Embeddable visualizations via iframe
- Live programming mode

---

## Source Code Structure

The repository is organized into multiple version directories, with **v5-unity** being the latest active version:

```
OnlinePythonTutor/
├── v3/                          # Legacy version with extensive documentation
│   ├── docs/                    # Developer documentation
│   ├── js/                      # JavaScript frontend (legacy)
│   ├── css/                     # Stylesheets
│   ├── pg_logger.py            # Python backend logger
│   ├── pg_encoder.py           # JSON trace encoder
│   ├── bottle_server.py        # Web server
│   └── web_exec*.py            # Language execution backends
│
├── v4-cokapi/                  # Non-Python language backends
│   ├── backends/
│   │   ├── c_cpp/              # C/C++ execution backend
│   │   ├── java/               # Java execution backend
│   │   ├── javascript/         # JavaScript execution backend
│   │   ├── ruby/               # Ruby execution backend
│   │   └── python-anaconda/    # Python with Anaconda
│   └── cokapi.js               # Node.js server for backends
│
├── v5-unity/                   # ⭐ CURRENT/ACTIVE VERSION
│   ├── js/                     # TypeScript frontend sources
│   │   ├── pytutor.ts          # Main visualization engine (~3911 lines)
│   │   ├── opt-frontend.ts     # Frontend UI controller (~757 lines)
│   │   ├── opt-frontend-common.ts
│   │   ├── opt-live.ts         # Live programming mode
│   │   └── lib/                # Third-party libraries
│   ├── css/                    # Stylesheets
│   ├── pg_logger.py            # Python backend (~1696 lines)
│   ├── pg_encoder.py           # JSON encoder (~545 lines)
│   ├── bottle_server.py        # Bottle web server
│   ├── web_exec*.py            # Backend executors per language
│   ├── webpack.config.js       # Webpack build configuration
│   ├── tsconfig.json           # TypeScript configuration
│   ├── package.json            # Node.js dependencies
│   ├── visualize.html          # Main visualization page
│   ├── live.html               # Live programming page
│   └── index.html              # Landing page
│
├── tests/                      # Test suites
├── requirements.txt            # Python dependencies
├── runtime.txt                 # Python runtime version
└── Procfile                    # Heroku deployment config
```

### Key File Purposes

#### Backend (Python)
- **pg_logger.py**: Core execution tracer using Python's `bdb` debugger
- **pg_encoder.py**: Encodes program state into JSON format
- **web_exec_*.py**: Language-specific execution wrappers
- **bottle_server.py**: Lightweight web server

#### Frontend (TypeScript/JavaScript)
- **pytutor.ts**: Main visualization engine (ExecutionVisualizer class)
- **opt-frontend.ts**: UI controls, editor integration, AJAX handlers
- **opt-frontend-common.ts**: Shared utilities and configuration
- **opt-live.ts**: Live collaborative programming features

---

## Data Flow Architecture

### High-Level Flow

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│   Browser   │ ──────> │   Frontend   │ ──────> │   Backend   │
│  (User UI)  │ <────── │  (HTML/TS/JS)│ <────── │  (Python)   │
└─────────────┘         └──────────────┘         └─────────────┘
     │                        │                         │
     │ 1. User types code     │                         │
     │ 2. Clicks "Visualize"  │                         │
     │                        │ 3. AJAX/JSONP request   │
     │                        │    with code string     │
     │                        │                         │
     │                        │                    4. Execute code
     │                        │                    5. Generate trace
     │                        │                         │
     │                        │ <───────────────────────┘
     │                        │  6. JSON trace response
     │                        │                         
     │ <──────────────────────┘                         
     │   7. Render visualization                        
     │   8. User steps through                          
```

### Detailed Execution Flow

#### 1. **User Interaction (Frontend)**
- User visits `visualize.html`
- Types code in Ace Editor (code editor component)
- Selects language (Python 2/3, Java, JavaScript, etc.)
- Clicks "Visualize Execution" button

#### 2. **Frontend Request Preparation**
Location: `v5-unity/js/opt-frontend.ts`

```typescript
// Simplified flow
function executeCode() {
  const code = editor.getValue();
  const language = selectedLanguage;
  
  // Make AJAX/JSONP request
  $.ajax({
    url: backendEndpoint,
    jsonp: "callback",
    dataType: "jsonp",
    data: {
      user_script: code,
      options_json: JSON.stringify(options)
    },
    success: handleResponse
  });
}
```

#### 3. **Backend Execution (Python)**
Location: `v5-unity/pg_logger.py`

```python
# Simplified flow
def exec_script_str(script_str, cumulative_mode, finalizer_func):
    # Create logger instance (subclass of bdb.Bdb debugger)
    logger = PGLogger(cumulative_mode, finalizer_func)
    
    try:
        # Execute user's code under debugger supervision
        logger._runscript(script_str)
    except bdb.BdbQuit:
        pass
    finally:
        # Generate final trace
        logger.finalize()
```

**Execution Steps:**
1. Create sandboxed environment with restricted builtins
2. Redirect stdout to capture print statements
3. Use `bdb` debugger to step through code line-by-line
4. At each step, capture:
   - Current line number
   - Stack frames
   - Variable values (globals and locals)
   - Heap objects (lists, dicts, objects, etc.)
   - stdout buffer contents

#### 4. **Trace Generation**
Location: `v5-unity/pg_encoder.py`

The backend generates a JSON trace with this structure:

```json
{
  "code": "x = 5\ny = 10\nz = x + y",
  "trace": [
    {
      "line": 1,
      "event": "step_line",
      "func_name": "<module>",
      "globals": {},
      "ordered_globals": [],
      "stack_to_render": [],
      "heap": {},
      "stdout": ""
    },
    {
      "line": 2,
      "event": "step_line",
      "func_name": "<module>",
      "globals": {"x": 5},
      "ordered_globals": ["x"],
      "stack_to_render": [],
      "heap": {},
      "stdout": ""
    }
    // ... more trace points
  ]
}
```

**Trace Entry Fields:**
- `line`: Line number about to execute
- `event`: Type of event (step_line, call, return, exception)
- `func_name`: Current function name
- `globals`: Global variable mappings
- `ordered_globals`: Order of variable appearance
- `stack_to_render`: List of stack frames
- `heap`: Heap objects (referenced by ID)
- `stdout`: Cumulative stdout content

#### 5. **Frontend Visualization**
Location: `v5-unity/js/pytutor.ts`

```typescript
class ExecutionVisualizer {
  constructor(domRoot, trace, params) {
    this.curInstr = 0;  // Current step
    this.trace = trace;  // Full execution trace
    // Initialize visualization
    this.renderStep(0);
  }
  
  renderStep(stepNum) {
    const entry = this.trace[stepNum];
    
    // Render code with current line highlighted
    this.renderPyCodeOutput(entry);
    
    // Render stack frames
    this.renderStackFrames(entry.stack_to_render);
    
    // Render heap objects
    this.renderHeapObjects(entry.heap);
    
    // Render globals
    this.renderGlobals(entry.globals);
    
    // Draw pointer connections using jsPlumb
    this.drawConnections();
  }
}
```

#### 6. **User Stepping Through Execution**
- User clicks "Forward" or "Back" buttons
- Frontend increments/decrements `curInstr` pointer
- Calls `renderStep(curInstr)` with new index
- **No backend communication needed** - all data is in the initial trace

---

## Technology Stack

### Backend Stack

#### Core Technologies
| Technology | Version/Type | Purpose |
|------------|--------------|---------|
| **Python** | 2.7 / 3.x | Primary backend language |
| **bdb** | stdlib | Python debugger for execution tracing |
| **Bottle** | 0.12.17 | Lightweight web framework |
| **Gunicorn** | 19.9.0 | WSGI HTTP server (production) |
| **Docker** | Latest | Sandboxing for non-Python backends |
| **Node.js** | v6.9.5 | Backend server for v4-cokapi |

#### Language-Specific Backends (v4-cokapi)
- **C/C++**: Custom Docker containers with GCC
- **Java**: JVM-based execution tracing
- **JavaScript**: Node.js execution
- **Ruby**: Ruby interpreter
- **TypeScript**: TypeScript compiler + Node.js

### Frontend Stack

#### Core Technologies
| Technology | Version | Purpose |
|------------|---------|---------|
| **TypeScript** | 2.8.3+ | Type-safe JavaScript development |
| **Webpack** | 3.11.0 | Module bundler and build tool |
| **jQuery** | 3.0.0 | DOM manipulation and AJAX |
| **D3.js** | v2 (legacy) | Data-driven DOM manipulation |
| **jsPlumb** | 1.3.10 | Drawing pointer connections |
| **Ace Editor** | Latest | Code editor component |
| **jQuery UI** | 1.11.4 | UI widgets and interactions |

#### Build Tools
- **ts-loader** (3.5.0): TypeScript compilation in Webpack
- **css-loader**: CSS module loading
- **style-loader**: CSS injection
- **url-loader**: Asset handling

#### Testing & Quality
- **Puppeteer** (1.5.0): Headless browser testing
- **Pixelmatch** (4.0.2): Visual regression testing

---

## Modern Stack Comparison

### Backend: Current vs. Modern

| Aspect | Current (OPT) | Modern Alternative | Gap Analysis |
|--------|---------------|-------------------|--------------|
| **Web Framework** | Bottle 0.12.17 (2017) | FastAPI, Flask 2.x+ | Bottle lacks async support, modern routing, OpenAPI |
| **Server** | Gunicorn 19.9.0 (2019) | Uvicorn, Hypercorn | Missing async/WebSocket support |
| **Python Version** | 2.7 & 3.x | Python 3.11+ | Python 2.7 EOL, missing modern features |
| **Sandboxing** | Custom + Docker | Docker + Kubernetes | Could benefit from orchestration |
| **API Style** | GET/JSONP | REST + WebSocket | No real-time bidirectional communication |
| **Type Safety** | None (plain Python) | Type hints + Pydantic | No runtime validation |
| **Debugging** | bdb (stdlib) | bdb + custom tracers | bdb is solid but inflexible |

**Key Gaps:**
- ❌ No async/await support for concurrent execution
- ❌ JSONP used for cross-domain (outdated, CORS is standard)
- ❌ No WebSocket support for real-time collaboration
- ❌ Python 2 support maintenance burden
- ❌ No structured API documentation (OpenAPI/Swagger)

### Frontend: Current vs. Modern

| Aspect | Current (OPT) | Modern Alternative | Gap Analysis |
|--------|---------------|-------------------|--------------|
| **Language** | TypeScript 2.8.3 | TypeScript 5.x | Missing newer TS features |
| **Module System** | Webpack 3.11.0 | Vite, Webpack 5, esbuild | Slow build times |
| **UI Framework** | jQuery + vanilla | React, Vue, Svelte | Manual DOM manipulation |
| **State Management** | Global variables | Redux, Zustand, Pinia | No centralized state |
| **UI Components** | jQuery UI | MUI, Ant Design, shadcn | Outdated component library |
| **Visualization** | D3 v2 + jsPlumb 1.3.10 | D3 v7, Canvas API, WebGL | Performance bottleneck |
| **Code Editor** | Ace Editor | Monaco (VSCode), CodeMirror 6 | Less feature-rich |
| **Styling** | Plain CSS | Tailwind, CSS Modules, styled-components | No utility-first approach |
| **Testing** | Manual + Puppeteer | Vitest, Playwright, Cypress | Limited test coverage |

**Key Gaps:**
- ❌ Component-based architecture would improve maintainability
- ❌ Virtual DOM would improve rendering performance
- ❌ Modern build tools would speed up development
- ❌ Deprecated D3 v2 API (v7 has breaking changes)
- ❌ jsPlumb 1.3.10 is very old (newer versions exist)
- ❌ No mobile-responsive design system
- ❌ Limited accessibility (ARIA) support

### Development Experience

| Aspect | Current | Modern | Gap |
|--------|---------|--------|-----|
| **Package Management** | npm + pip | pnpm/yarn + poetry/pipenv | Slower installs |
| **Linting** | Limited | ESLint, Prettier, Black | Inconsistent style |
| **Hot Reload** | Webpack watch | Vite HMR | Slower feedback |
| **Deployment** | Heroku (Procfile) | Vercel, Railway, Cloud Run | Limited CI/CD |
| **Monitoring** | None | Sentry, DataDog, Posthog | No error tracking |
| **Documentation** | Markdown | Docusaurus, VitePress | No interactive docs |

---

## VSCode Plugin Feasibility

### Why VSCode Extension?

**Benefits:**
1. ✅ Native IDE integration
2. ✅ Access to debugging API
3. ✅ Local execution (no server required)
4. ✅ Better developer experience
5. ✅ Offline capability
6. ✅ Leverage existing VSCode debugging infrastructure

### Architecture Proposal

```
┌─────────────────────────────────────────────────────────┐
│                    VSCode Extension                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   UI Panel   │  │   Debugger   │  │    Trace     │ │
│  │  (Webview)   │  │   Adapter    │  │  Generator   │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│         │                 │                  │          │
│         └─────────────────┴──────────────────┘          │
│                          │                              │
└──────────────────────────┼──────────────────────────────┘
                           │
                ┌──────────┴──────────┐
                │  VSCode Debug API   │
                └──────────┬──────────┘
                           │
                ┌──────────┴──────────┐
                │  Language Runtime   │
                │  (Python, Node.js)  │
                └─────────────────────┘
```

### Technical Approach

#### Option 1: Pure VSCode Debug Adapter (Recommended)

**Pros:**
- ✅ Leverages existing VSCode debugging infrastructure
- ✅ Language-agnostic (works with Python, JavaScript, etc.)
- ✅ Reuses existing debug protocols (DAP)
- ✅ No need to replicate execution tracing

**Cons:**
- ⚠️ Limited to languages with DAP support
- ⚠️ Requires custom Debug Adapter Protocol implementation
- ⚠️ Complex state synchronization

**Implementation:**
```typescript
// Extension entry point
export function activate(context: vscode.ExtensionContext) {
  // Register debug adapter
  context.subscriptions.push(
    vscode.debug.registerDebugAdapterDescriptorFactory(
      'pythontutor', 
      new PythonTutorDebugAdapterFactory()
    )
  );
  
  // Register visualization panel
  context.subscriptions.push(
    vscode.commands.registerCommand(
      'pythontutor.visualize',
      () => {
        const panel = vscode.window.createWebviewPanel(
          'pythontutor',
          'Python Tutor Visualization',
          vscode.ViewColumn.Two,
          { enableScripts: true }
        );
        
        // Start debug session
        vscode.debug.startDebugging(
          undefined,
          {
            type: 'pythontutor',
            name: 'Python Tutor',
            request: 'launch',
            program: vscode.window.activeTextEditor?.document.fileName
          }
        );
        
        // Listen to debug events
        vscode.debug.onDidChangeBreakpoints(e => {
          // Update visualization on step
        });
      }
    )
  );
}
```

#### Option 2: Embedded Python Backend

**Pros:**
- ✅ Full control over execution tracing
- ✅ Can reuse existing `pg_logger.py` code
- ✅ Same trace format as web version

**Cons:**
- ⚠️ Requires Python runtime
- ⚠️ Complex installation process
- ⚠️ Cross-platform challenges

**Implementation:**
```typescript
// Run Python backend as child process
import { spawn } from 'child_process';

function generateTrace(code: string): Promise<Trace> {
  return new Promise((resolve, reject) => {
    const python = spawn('python', [
      '-c',
      `import pg_logger; pg_logger.exec_script_str('''${code}''', ...)`
    ]);
    
    let output = '';
    python.stdout.on('data', data => output += data);
    python.on('close', () => resolve(JSON.parse(output)));
  });
}
```

#### Option 3: Hybrid Approach (Best)

Combine both approaches:
1. Use VSCode Debug API for stepping/breakpoints
2. Extract trace data from debug session
3. Convert to OPT trace format
4. Render using ported visualization code

**Benefits:**
- ✅ Native VSCode integration
- ✅ No external dependencies
- ✅ Language support via Debug Adapters
- ✅ Familiar debugging workflow

### Component Breakdown

#### 1. Extension Manifest (`package.json`)
```json
{
  "name": "pythontutor-vscode",
  "displayName": "Python Tutor Visualizer",
  "version": "1.0.0",
  "engines": {
    "vscode": "^1.80.0"
  },
  "activationEvents": [
    "onDebug",
    "onCommand:pythontutor.visualize"
  ],
  "main": "./out/extension.js",
  "contributes": {
    "commands": [
      {
        "command": "pythontutor.visualize",
        "title": "Visualize with Python Tutor"
      }
    ],
    "debuggers": [
      {
        "type": "pythontutor",
        "label": "Python Tutor Debugger"
      }
    ]
  }
}
```

#### 2. Debug Adapter Implementation
```typescript
class PythonTutorDebugSession extends DebugSession {
  private trace: ExecutionTrace = [];
  
  protected initializeRequest(response, args): void {
    // Set up debug capabilities
  }
  
  protected launchRequest(response, args): void {
    // Start debugging session
    this.attachToDebugger(args.program);
  }
  
  protected nextRequest(response): void {
    // Step to next line and capture state
    this.captureExecutionState();
    this.sendEvent(new StoppedEvent('step', 1));
  }
  
  private captureExecutionState(): void {
    // Read current stack, variables, heap
    const state = {
      line: this.currentLine,
      stack: this.getStackFrames(),
      globals: this.getGlobalVariables(),
      heap: this.getHeapObjects()
    };
    this.trace.push(state);
    
    // Send to webview
    this.sendToWebview(state);
  }
}
```

#### 3. Webview Visualization
```typescript
class VisualizationPanel {
  private panel: vscode.WebviewPanel;
  
  constructor(extensionUri: vscode.Uri) {
    this.panel = vscode.window.createWebviewPanel(
      'pythontutor',
      'Python Tutor',
      vscode.ViewColumn.Two,
      {
        enableScripts: true,
        localResourceRoots: [extensionUri]
      }
    );
    
    // Load visualization HTML/JS from OPT
    this.panel.webview.html = this.getWebviewContent();
    
    // Handle messages from webview
    this.panel.webview.onDidReceiveMessage(msg => {
      switch (msg.command) {
        case 'step':
          vscode.debug.activeDebugSession?.customRequest('next');
          break;
      }
    });
  }
  
  private getWebviewContent(): string {
    // Port pytutor.ts visualization code
    // Or embed as iframe to local server
    return `
      <!DOCTYPE html>
      <html>
        <head>
          <link rel="stylesheet" href="${pytutorCss}">
        </head>
        <body>
          <div id="visualization"></div>
          <script src="${pytutorJs}"></script>
        </body>
      </html>
    `;
  }
}
```

### Implementation Roadmap

#### Phase 1: Proof of Concept (2-3 weeks)
- [ ] Set up VSCode extension boilerplate
- [ ] Implement basic Debug Adapter Protocol integration
- [ ] Create simple webview with hardcoded trace
- [ ] Test with Python debugger

#### Phase 2: Core Features (4-6 weeks)
- [ ] Port trace generation logic from `pg_logger.py`
- [ ] Implement state capture at each debug step
- [ ] Port visualization code from `pytutor.ts` to webview
- [ ] Add step controls (forward/back)
- [ ] Render stack frames and variables

#### Phase 3: Advanced Features (4-6 weeks)
- [ ] Heap object visualization with pointer arrows
- [ ] Support for complex data structures
- [ ] Multiple language support (JavaScript, etc.)
- [ ] Breakpoint integration
- [ ] Export/share trace functionality

#### Phase 4: Polish & Release (2-3 weeks)
- [ ] UI/UX improvements
- [ ] Documentation
- [ ] Testing
- [ ] Marketplace submission

**Total Estimated Effort**: 12-18 weeks (3-4.5 months)

### Technical Challenges

1. **State Synchronization**
   - Challenge: Keep debug session state in sync with visualization
   - Solution: Use message passing between debug adapter and webview

2. **Language Support**
   - Challenge: Each language has different debug protocol
   - Solution: Abstract debug adapter interface, implement per language

3. **Performance**
   - Challenge: Large traces may slow down visualization
   - Solution: Virtualize rendering, lazy load heap objects

4. **Heap Object Tracking**
   - Challenge: VSCode debug API doesn't expose object IDs
   - Solution: Use memory addresses or implement custom object tracking

5. **Cross-Platform**
   - Challenge: Different debuggers on Windows/Mac/Linux
   - Solution: Use Debug Adapter Protocol (standardized)

### Competitive Analysis

**Existing Similar Extensions:**
- **Python Visualizer**: Basic variable inspection
- **Code Runner**: Executes code but no visualization
- **Debugging Visualizer**: Shows debug state but not step-by-step

**Advantages of Python Tutor VSCode Extension:**
- ✅ Superior visualization (proven UX from web version)
- ✅ Multi-language support
- ✅ Educational focus (not just debugging)
- ✅ Embeddable in other extensions

---

## Key Components Deep Dive

### Backend: `pg_logger.py`

**Core Class: PGLogger**
```python
class PGLogger(bdb.Bdb):
    """
    Subclass of bdb.Bdb (Python debugger).
    Intercepts execution at each step to build trace.
    """
    
    def __init__(self, cumulative_mode, finalizer_func):
        bdb.Bdb.__init__(self)
        self.trace = []  # List of execution points
        self.cumulative_mode = cumulative_mode
        self.finalizer_func = finalizer_func
        
    def _runscript(self, script_str):
        # Set up sandboxed environment
        user_builtins = get_safe_builtins()
        user_stdout = StringIO.StringIO()
        user_globals = {
            '__builtins__': user_builtins,
            '__name__': '__main__'
        }
        
        # Execute under debugger supervision
        self.run(script_str, user_globals, user_globals)
    
    def interaction(self, frame, event_type):
        """
        Called at every step (line, call, return, exception).
        Captures execution state and appends to trace.
        """
        # Build trace entry
        trace_entry = {
            'line': frame.f_lineno,
            'event': event_type,
            'func_name': frame.f_code.co_name,
            'globals': self.encode_globals(frame),
            'stack_to_render': self.encode_stack(frame),
            'heap': self.heap,
            'stdout': self.stdout_buffer.getvalue()
        }
        
        self.trace.append(trace_entry)
        
        # Guard against infinite loops
        if len(self.trace) > MAX_EXECUTED_LINES:
            raise bdb.BdbQuit
```

**Key Methods:**
- `user_line()`: Called at each line execution
- `user_call()`: Called at function entry
- `user_return()`: Called at function return
- `user_exception()`: Called on exception

### Frontend: `pytutor.ts`

**Core Class: ExecutionVisualizer**
```typescript
export class ExecutionVisualizer {
  curInstr: number = 0;  // Current step index
  trace: ExecutionTrace;  // Full execution trace
  domRoot: HTMLElement;   // Root DOM element
  
  constructor(domRoot, trace, params) {
    this.domRoot = domRoot;
    this.trace = trace;
    this.params = params;
    
    // Initialize UI
    this.renderAllOutput();
  }
  
  renderStep(stepNum: number) {
    const entry = this.trace[stepNum];
    
    // 1. Highlight current line in code
    this.renderPyCodeOutput();
    
    // 2. Render stack frames
    this.renderStackFrames(entry.stack_to_render);
    
    // 3. Render global variables
    this.updateOutput();
    
    // 4. Render heap objects
    this.renderHeapObjects();
    
    // 5. Draw arrows between references
    this.drawConnections();
  }
  
  renderHeapObjects() {
    const heap = this.curTrace.heap;
    
    for (const [objId, objData] of Object.entries(heap)) {
      const type = objData[0];  // 'LIST', 'DICT', 'INSTANCE', etc.
      
      // Create DOM element based on type
      switch (type) {
        case 'LIST':
          this.renderList(objId, objData);
          break;
        case 'DICT':
          this.renderDict(objId, objData);
          break;
        case 'INSTANCE':
          this.renderInstance(objId, objData);
          break;
      }
    }
  }
  
  drawConnections() {
    // Use jsPlumb to draw arrows from variables to heap objects
    jsPlumb.connect({
      source: variableDiv,
      target: heapObjectDiv,
      connector: ['Straight'],
      endpoint: ['Dot', { radius: 3 }],
      paintStyle: { stroke: '#005583', strokeWidth: 2 }
    });
  }
}
```

**Rendering Pipeline:**
1. Parse trace entry
2. Create/update DOM elements for:
   - Code editor with highlighted line
   - Stack frames
   - Global variables
   - Heap objects (lists, dicts, objects)
3. Use jsPlumb to draw pointer connections
4. Handle user interactions (step forward/back)

### Trace Encoder: `pg_encoder.py`

**Object Encoding Examples:**
```python
# Primitives (inline)
encode(5)        # → 5
encode("hello")  # → "hello"
encode(True)     # → true
encode(None)     # → null

# Compound types (heap references)
encode([1, 2, 3])              # → ["REF", 1]
# heap[1] = ["LIST", 1, 2, 3]

encode({"a": 1, "b": 2})       # → ["REF", 2]
# heap[2] = ["DICT", ["a", 1], ["b", 2]]

encode(MyClass())              # → ["REF", 3]
# heap[3] = ["INSTANCE", "MyClass", [...]]

# Nested references
encode([1, {"a": 2}])          # → ["REF", 4]
# heap[4] = ["LIST", 1, ["REF", 2]]
# heap[2] = ["DICT", ["a", 2]]
```

**Heap Object Formats:**
- `["LIST", item1, item2, ...]`
- `["TUPLE", item1, item2, ...]`
- `["DICT", [key1, val1], [key2, val2], ...]`
- `["SET", item1, item2, ...]`
- `["INSTANCE", className, [["attr1", val1], ["attr2", val2]]]`
- `["FUNCTION", "funcName(args)", parentFrameId]`

---

## Development Workflow

### Local Development Setup

#### Backend (Python)
```bash
# Install dependencies
pip install bottle gunicorn

# Run development server
cd v5-unity/
python bottle_server.py

# Access at http://localhost:8003/visualize.html
```

#### Frontend (TypeScript)
```bash
# Install dependencies
npm install

# Watch mode (auto-recompile)
npm run webpack

# Production build
npm run production-build
```

### Testing

#### Backend Tests
```bash
cd v3/tests/
./run-all-tests.sh

# Run specific test
python golden_test.py --test=<test_name>
```

#### Frontend Tests
- Manual testing via `demo.html`
- Visual regression tests with Puppeteer
- Limited automated test coverage

### Deployment

#### Heroku (Current)
```bash
git push heroku master
```

Configuration files:
- `Procfile`: Defines web server command
- `runtime.txt`: Python version
- `requirements.txt`: Dependencies

#### Alternative Deployment Options
- **Vercel**: Frontend static hosting + serverless functions
- **Docker**: Full containerized deployment
- **AWS Lambda**: Serverless backend
- **Google Cloud Run**: Container-based serverless

---

## Conclusion

Online Python Tutor is a mature educational tool with a well-designed architecture separating backend execution from frontend visualization. The trace-based approach enables rich step-by-step debugging without continuous server communication.

### Strengths
- ✅ Clean separation of concerns
- ✅ Language-agnostic trace format
- ✅ Embeddable visualizations
- ✅ Proven educational value

### Areas for Improvement
- ⚠️ Outdated dependencies (Webpack 3, D3 v2, jsPlumb 1.3.10)
- ⚠️ Python 2 legacy support
- ⚠️ No component-based architecture
- ⚠️ Limited real-time collaboration features
- ⚠️ Performance bottlenecks with large traces

### VSCode Extension Potential
The VSCode extension is **highly feasible** with an estimated 3-4.5 month development timeline. The hybrid approach leveraging Debug Adapter Protocol while reusing visualization code provides the best balance of effort and functionality.

---

**Document Version**: 1.0  
**Last Updated**: October 2025  
**Author**: Technical Documentation Team  
**Contact**: For questions, see repository README.md
