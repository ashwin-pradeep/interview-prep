# Vite vs Webpack: Deep Comparison, Use Cases & Bundler Guide

> **Target Audience:** Senior Front End Developer Interview Preparation

## Table of Contents

1. [Vite Deep Dive](#vite-deep-dive)
2. [Webpack Deep Dive](#webpack-deep-dive)
3. [Vite vs Webpack Head-to-Head Comparison](#vite-vs-webpack-head-to-head-comparison)
4. [Other Bundlers Quick Comparison](#other-bundlers-quick-comparison)
5. [Practical Scenarios](#practical-scenarios)

---

## Vite Deep Dive

### How does Vite work internally?

Vite has two distinct modes:

**Development mode:**
- Serves source files as native ES Modules (no bundling)
- Uses **esbuild** to pre-bundle dependencies (100x faster than webpack)
- Browser requests individual modules on demand
- Instant server start regardless of project size

**Production mode:**
- Uses **Rollup** to create optimized bundles
- Tree-shaking, code splitting, minification
- Output is standard bundled JS/CSS

```
Dev mode:
Browser → Vite Dev Server → Transforms module on demand → Returns ES Module
               ↑
         esbuild pre-bundled deps (node_modules → single ESM file)

Prod mode:
Source Files → Rollup → Bundle → dist/
```

---

### Why is Vite so fast in development?

**Key insight:** Traditional bundlers like webpack must process *all* modules before the browser sees anything. Vite doesn't bundle at all during development.

| Reason | Explanation |
|---|---|
| **No dev-time bundling** | Browser natively handles ES module imports |
| **On-demand transforms** | Only requested modules are compiled |
| **esbuild for deps** | Pre-bundling with esbuild is 10–100x faster than babel/tsc |
| **Efficient HMR** | Updates only the changed module, not the whole bundle |
| **Caching** | Pre-bundled deps cached aggressively |

**esbuild is fast because:**
- Written in Go (not JavaScript)
- Parallelizes work across all CPU cores
- Zero use of AST-to-string conversions

---

### How does Vite's Dev Server architecture work?

```
Request: /src/App.tsx
    ↓
Vite Middleware Pipeline:
  1. Check cache → hit? return cached
  2. Transform request:
     - .ts/.tsx → esbuild transform → JS
     - .vue/.svelte → framework plugin → JS
     - .css → CSS transform → JS module with CSS injection
  3. Inject HMR client code
  4. Return module
```

---

### How does Vite HMR differ from Webpack HMR?

| Feature | Vite HMR | Webpack HMR |
|---|---|---|
| **Granularity** | Module-level — only changed module replaced | Can be full chunk replacement |
| **Speed** | Instant (< 50ms typically) | Seconds (incremental rebuild) |
| **CSS HMR** | Instant — no page reload | Depends on setup |
| **State preservation** | Yes (React Fast Refresh) | Yes (requires react-hot-loader) |
| **How it works** | WebSocket → browser fetches updated module | WebSocket → runtime patches chunk |

```js
// Vite HMR API (for custom modules)
if (import.meta.hot) {
  import.meta.hot.accept('./module.js', (newModule) => {
    // handle module update
  });
  
  import.meta.hot.dispose((data) => {
    // cleanup before module is replaced
  });
}
```

---

### How does the Vite plugin system work?

Vite plugins are a superset of Rollup plugins — most Rollup plugins work directly in Vite.

```ts
// vite.config.ts — custom plugin example
import { defineConfig, Plugin } from 'vite';

const myPlugin = (): Plugin => ({
  name: 'my-plugin',
  
  // Rollup hooks (work in both dev and build)
  resolveId(id) {
    if (id === 'virtual:my-module') return id;
  },
  
  load(id) {
    if (id === 'virtual:my-module') {
      return 'export const greeting = "Hello from virtual module!";';
    }
  },
  
  transform(code, id) {
    if (id.endsWith('.special')) {
      return code.replace('REPLACE_ME', 'replaced value');
    }
  },
  
  // Vite-specific hooks
  configureServer(server) {
    server.middlewares.use('/custom', (req, res) => {
      res.end('Custom middleware response');
    });
  },
  
  handleHotUpdate({ file, server }) {
    if (file.endsWith('.data')) {
      server.ws.send({ type: 'full-reload' });
    }
  },
});
```

---

### What does a complete vite.config.ts look like?

```typescript
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig(({ command, mode }) => {
  const env = loadEnv(mode, process.cwd(), '');
  
  return {
    plugins: [react()],
    
    // Module aliases
    resolve: {
      alias: {
        '@': path.resolve(__dirname, './src'),
        '@components': path.resolve(__dirname, './src/components'),
        '@hooks': path.resolve(__dirname, './src/hooks'),
        '@utils': path.resolve(__dirname, './src/utils'),
      },
    },
    
    // Dev server configuration
    server: {
      port: 3000,
      open: true,
      proxy: {
        '/api': {
          target: 'http://localhost:8080',
          changeOrigin: true,
          rewrite: (path) => path.replace(/^\/api/, ''),
        },
      },
      cors: true,
    },
    
    // Build configuration
    build: {
      outDir: 'dist',
      sourcemap: mode === 'development',
      minify: 'esbuild',
      target: 'es2020',
      
      rollupOptions: {
        output: {
          // Manual chunk splitting
          manualChunks: {
            'react-vendor': ['react', 'react-dom'],
            'router': ['react-router-dom'],
            'query': ['@tanstack/react-query'],
          },
        },
      },
      
      // Chunk size warning threshold
      chunkSizeWarningLimit: 600,
    },
    
    // CSS configuration
    css: {
      modules: {
        localsConvention: 'camelCaseOnly',
      },
      preprocessorOptions: {
        scss: {
          additionalData: `@use "@/styles/variables" as *;`,
        },
      },
    },
    
    // Environment variable prefix
    envPrefix: 'VITE_',
    
    // Preview server (for testing production build locally)
    preview: {
      port: 4173,
    },
    
    // Test configuration (vitest)
    test: {
      globals: true,
      environment: 'jsdom',
      setupFiles: './src/test/setup.ts',
    },
  };
});
```

---

### How do Environment Variables work in Vite?

```bash
# .env files (loaded in priority order)
.env                 # always loaded
.env.local           # always loaded, git-ignored
.env.development     # loaded in dev mode
.env.production      # loaded in prod mode
.env.staging         # custom mode
```

```bash
# Only variables prefixed with VITE_ are exposed to client
VITE_API_URL=https://api.example.com
VITE_APP_NAME=MyApp
SECRET_KEY=not-exposed   # server-only, not accessible in browser
```

```tsx
// Accessing in code
const apiUrl = import.meta.env.VITE_API_URL;
const isDev = import.meta.env.DEV;       // boolean
const isProd = import.meta.env.PROD;     // boolean
const mode = import.meta.env.MODE;       // 'development' | 'production' | custom
const baseUrl = import.meta.env.BASE_URL; // base URL from config
```

```typescript
// Type safety for env vars
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_NAME: string;
  readonly VITE_FEATURE_FLAG: 'true' | 'false';
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

### How does Vite support SSR?

```ts
// vite.config.ts - SSR configuration
export default defineConfig({
  build: {
    ssr: true,           // enable SSR mode
    rollupOptions: {
      input: 'src/entry-server.tsx',
    },
  },
});
```

```tsx
// entry-server.tsx
import { renderToString } from 'react-dom/server';
import App from './App';

export function render(url: string) {
  const html = renderToString(<App url={url} />);
  return { html };
}
```

---

### What is Vite Library Mode?

```ts
// vite.config.ts for publishing a library
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  build: {
    lib: {
      entry: resolve(__dirname, 'src/index.ts'),
      name: 'MyLib',
      fileName: 'my-lib',
      formats: ['es', 'cjs', 'umd'],
    },
    rollupOptions: {
      // Don't bundle peer dependencies
      external: ['react', 'react-dom'],
      output: {
        globals: {
          react: 'React',
          'react-dom': 'ReactDOM',
        },
      },
    },
  },
});
```

---

## Webpack Deep Dive

### How does Webpack work internally?

Webpack builds a **dependency graph** starting from entry points:

```
Entry Point (src/index.js)
    ↓
Webpack reads file
    ↓
Parse for imports/requires
    ↓
Resolve dependencies → add to graph
    ↓
Apply loaders to each module (transpile, transform)
    ↓
Apply plugins (optimize, inject, copy)
    ↓
Output bundles to disk
```

**Key concepts:**
- **Module:** Any file Webpack processes (JS, CSS, images, etc.)
- **Chunk:** Group of modules compiled together
- **Bundle:** Output file(s) produced from chunks

---

### What are Webpack Loaders and how do they work?

Loaders transform files before they're added to the dependency graph. They're chained (applied right-to-left):

```js
// webpack.config.js - common loaders
module.exports = {
  module: {
    rules: [
      // Transpile TypeScript and modern JS
      {
        test: /\.(ts|tsx|js|jsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env', '@babel/preset-react', '@babel/preset-typescript'],
          },
        },
      },
      
      // CSS processing (applied right-to-left)
      {
        test: /\.css$/,
        use: [
          'style-loader',    // 3. Injects CSS into DOM
          'css-loader',      // 2. Resolves @import and url()
          'postcss-loader',  // 1. Processes with PostCSS (autoprefixer)
        ],
      },
      
      // CSS Modules
      {
        test: /\.module\.css$/,
        use: [
          'style-loader',
          {
            loader: 'css-loader',
            options: { modules: { localIdentName: '[name]__[local]--[hash:base64:5]' } },
          },
        ],
      },
      
      // Images
      {
        test: /\.(png|jpg|gif|svg|webp)$/,
        type: 'asset',        // Webpack 5: replaces file-loader/url-loader
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024, // inline if < 8KB
          },
        },
      },
    ],
  },
};
```

---

### What are key Webpack Plugins and what do they do?

```js
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const { DefinePlugin, BannerPlugin } = require('webpack');
const CopyPlugin = require('copy-webpack-plugin');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  plugins: [
    // Generate HTML file with script tags injected
    new HtmlWebpackPlugin({
      template: './public/index.html',
      title: 'My App',
    }),
    
    // Extract CSS into separate files (production)
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash].css',
    }),
    
    // Replace global constants at compile time
    new DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
      'process.env.API_URL': JSON.stringify(process.env.API_URL),
      __DEV__: JSON.stringify(process.env.NODE_ENV !== 'production'),
    }),
    
    // Copy static assets
    new CopyPlugin({
      patterns: [{ from: 'public', to: 'dist', globOptions: { ignore: ['**/index.html'] } }],
    }),
    
    // Analyze bundle (open in browser after build)
    process.env.ANALYZE && new BundleAnalyzerPlugin(),
  ].filter(Boolean),
};
```

---

### How does Webpack Code Splitting work?

```js
// 1. Entry points code splitting
module.exports = {
  entry: {
    main: './src/index.js',
    vendor: './src/vendor.js',
  },
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
        common: {
          minChunks: 2,
          name: 'common',
          chunks: 'initial',
        },
      },
    },
  },
};
```

```js
// 2. Dynamic imports (async code splitting)
const loadDashboard = async () => {
  const { Dashboard } = await import('./Dashboard');
  return Dashboard;
};

// React.lazy uses dynamic import
const Dashboard = React.lazy(() => import('./Dashboard'));
```

---

### What is Webpack Module Federation?

Module Federation enables micro-frontends by allowing applications to share code at runtime.

```js
// Host (shell) application
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        // Load from remote app at runtime
        productApp: 'productApp@http://localhost:3001/remoteEntry.js',
        cartApp: 'cartApp@http://localhost:3002/remoteEntry.js',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18' },
        'react-dom': { singleton: true, requiredVersion: '^18' },
      },
    }),
  ],
};
```

```js
// Remote (product) application
new ModuleFederationPlugin({
  name: 'productApp',
  filename: 'remoteEntry.js',
  exposes: {
    './ProductList': './src/components/ProductList',
    './ProductDetail': './src/components/ProductDetail',
  },
  shared: {
    react: { singleton: true },
  },
});
```

```tsx
// Using remote module in host
const ProductList = React.lazy(() => import('productApp/ProductList'));
```

---

### How does Webpack optimization work for production?

```js
// webpack.config.js - production optimizations
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

module.exports = {
  mode: 'production',
  
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        parallel: true,      // use multiple CPU cores
        terserOptions: {
          compress: { drop_console: true }, // remove console.log in prod
        },
      }),
      new CssMinimizerPlugin(),
    ],
    
    // Content hash for long-term caching
    moduleIds: 'deterministic',
    chunkIds: 'deterministic',
    
    // Hoist modules into parent scope (tree shaking)
    concatenateModules: true,
    
    // Split chunks configuration
    splitChunks: {
      chunks: 'all',
      maxInitialRequests: 25,
      minSize: 20000,
      cacheGroups: {
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'react',
          chunks: 'all',
          priority: 20,
        },
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          priority: 10,
        },
      },
    },
    
    // Webpack 5: persistent caching
    runtimeChunk: 'single',
  },
  
  // Webpack 5 persistent cache
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename],
    },
  },
};
```

---

### What are Webpack 5's major new features?

| Feature | Description |
|---|---|
| **Module Federation** | Share modules between separately built apps |
| **Persistent Disk Cache** | Cache build results to filesystem (much faster rebuilds) |
| **Asset Modules** | Built-in file/url/raw loader (replaces file-loader, url-loader) |
| **Improved tree shaking** | Better side-effects detection, nested tree shaking |
| **WebAssembly** | Better WASM support, sync loading |
| **Deterministic module IDs** | Stable chunk hashes across builds |
| **Node.js polyfill removal** | No longer polyfills node.js core modules by default |

---

## Vite vs Webpack Head-to-Head Comparison

### Comprehensive Vite vs Webpack comparison

| Aspect | Vite | Webpack |
|---|---|---|
| **Dev server start** | Instant (~300ms) | Slow (seconds to minutes for large apps) |
| **HMR speed** | < 50ms typically | Seconds (incremental rebuild) |
| **Production build** | Fast (Rollup) | Slower (can use SWC for speed) |
| **Configuration** | Simple, minimal | Complex, highly configurable |
| **Learning curve** | Low | High |
| **Plugin ecosystem** | Good (growing) | Excellent (mature) |
| **Browser support** | Modern browsers (ESM) | Any browser |
| **SSR support** | Good (first-party) | Good (with config) |
| **Module Federation** | Via plugin (vite-plugin-federation) | First-class (built-in) |
| **Code splitting** | Automatic (Rollup) | Manual configuration required |
| **Tree shaking** | Excellent (Rollup) | Good (requires config) |
| **CSS handling** | Built-in | Requires loaders/plugins |
| **TypeScript** | Built-in (esbuild) | Requires ts-loader or babel |
| **Enterprise readiness** | Growing | Proven at scale |
| **Docker-friendly** | Yes | Yes |
| **CRA replacement** | Yes (drop-in via create-vite) | Somewhat |

---

### When should you choose Vite vs Webpack?

**Choose Vite when:**
- New project — greenfield development
- Modern browser support is acceptable (no IE11)
- Developer experience and speed are priorities
- React, Vue, or Svelte project
- Library development (library mode)
- Team is small to medium

**Choose Webpack when:**
- Legacy browser support required (IE11)
- Complex module customization needed
- Module Federation for micro-frontends
- Enterprise project with existing webpack setup
- Need very fine-grained control over bundling
- Plugin requirements not yet available in Vite ecosystem

---

### How do you migrate from Webpack to Vite?

```bash
# Step 1: Install Vite and plugin
npm install --save-dev vite @vitejs/plugin-react

# Step 2: Remove webpack dependencies
npm uninstall webpack webpack-cli webpack-dev-server \
  babel-loader css-loader style-loader html-webpack-plugin \
  @babel/core @babel/preset-env @babel/preset-react
```

```ts
// Step 3: Create vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      // Convert webpack aliases
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 3000,
  },
});
```

```html
<!-- Step 4: Update index.html (move to root, add type="module") -->
<!-- FROM: public/index.html (webpack) -->
<!-- TO:   index.html (project root) -->
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>  <!-- Add this -->
  </body>
</html>
```

```json
// Step 5: Update package.json scripts
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  }
}
```

```ts
// Step 6: Update env variables
// FROM: process.env.REACT_APP_API_URL
// TO:   import.meta.env.VITE_API_URL

// .env
// FROM: REACT_APP_API_URL=https://api.example.com
// TO:   VITE_API_URL=https://api.example.com
```

**Common migration issues:**
- `require()` not supported in ESM → replace with `import`
- `__dirname` not available → use `import.meta.url` + `fileURLToPath`
- CommonJS dependencies → Vite handles most via pre-bundling
- Global variables (process.env) → use `import.meta.env`

---

### How do you migrate from Create React App (CRA) to Vite?

```bash
# Step 1: Remove CRA
npm uninstall react-scripts

# Step 2: Install Vite
npm install --save-dev vite @vitejs/plugin-react

# Step 3: Install missing dependencies (CRA included these)
npm install --save-dev @types/react @types/react-dom typescript
```

```ts
// Step 4: Create vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: { port: 3000 },
});
```

```html
<!-- Step 5: Move public/index.html to root and update -->
<!-- Add to head: -->
<!-- Remove %PUBLIC_URL% references -->
<script type="module" src="/src/index.tsx"></script>
```

```typescript
// Step 6: Update tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx"
  },
  "include": ["src"]
}
```

```json
// Step 7: Update scripts
{
  "scripts": {
    "start": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest"  // replace react-scripts test
  }
}
```

---

### How do you set up a new React + TypeScript project with Vite?

```bash
# Scaffold with create-vite
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

```bash
# Or with additional packages
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm install react-router-dom @tanstack/react-query zustand
npm install --save-dev vitest @testing-library/react @testing-library/jest-dom jsdom
```

```ts
// Final vite.config.ts for a production React + TypeScript app
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: false, gzipSize: true }),
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 3000,
    proxy: {
      '/api': { target: 'http://localhost:8080', changeOrigin: true },
    },
  },
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['react-router-dom'],
        },
      },
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
  },
});
```

---

### How do you set up Webpack 5 for a new React + TypeScript project?

```bash
mkdir my-app && cd my-app
npm init -y
npm install react react-dom
npm install --save-dev \
  webpack webpack-cli webpack-dev-server \
  typescript ts-loader \
  html-webpack-plugin \
  css-loader style-loader \
  @types/react @types/react-dom
```

```js
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = (env, argv) => {
  const isDev = argv.mode === 'development';
  
  return {
    entry: './src/index.tsx',
    
    output: {
      path: path.resolve(__dirname, 'dist'),
      filename: '[name].[contenthash].js',
      clean: true,
      publicPath: '/',
    },
    
    resolve: {
      extensions: ['.tsx', '.ts', '.js'],
      alias: { '@': path.resolve(__dirname, 'src') },
    },
    
    module: {
      rules: [
        {
          test: /\.(ts|tsx)$/,
          use: 'ts-loader',
          exclude: /node_modules/,
        },
        {
          test: /\.css$/,
          use: ['style-loader', 'css-loader'],
        },
      ],
    },
    
    plugins: [
      new HtmlWebpackPlugin({ template: './public/index.html' }),
    ],
    
    devServer: {
      port: 3000,
      historyApiFallback: true,
      hot: true,
      proxy: {
        '/api': { target: 'http://localhost:8080', changeOrigin: true },
      },
    },
    
    devtool: isDev ? 'eval-source-map' : 'source-map',
    
    cache: {
      type: 'filesystem',
    },
  };
};
```

---

### How do you configure module aliases in Vite and Webpack?

```ts
// Vite — vite.config.ts
import path from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@hooks': path.resolve(__dirname, './src/hooks'),
    },
  },
});
```

```json
// TypeScript — tsconfig.json (same for both Vite and Webpack)
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@hooks/*": ["src/hooks/*"]
    }
  }
}
```

```js
// Webpack — webpack.config.js
const path = require('path');

module.exports = {
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@components': path.resolve(__dirname, 'src/components'),
      '@hooks': path.resolve(__dirname, 'src/hooks'),
    },
  },
};
```

---

### How do you configure proxy for API calls in Vite and Webpack?

```ts
// Vite proxy
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
        secure: false,
      },
      '/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
});
```

```js
// Webpack DevServer proxy
module.exports = {
  devServer: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        pathRewrite: { '^/api': '' },
        secure: false,
      },
    },
  },
};
```

---

## Other Bundlers Quick Comparison

### Rollup — When and why?

**Best for:** Libraries and packages (not applications)

```js
// rollup.config.js
export default {
  input: 'src/index.ts',
  output: [
    { file: 'dist/index.cjs.js', format: 'cjs' },
    { file: 'dist/index.esm.js', format: 'esm' },
    { file: 'dist/index.umd.js', format: 'umd', name: 'MyLib' },
  ],
  plugins: [
    resolve(),
    commonjs(),
    typescript(),
    terser(),
  ],
  external: ['react', 'react-dom'],
};
```

**Why Rollup for libraries:**
- Excellent tree shaking
- Clean, minimal output
- Multiple output formats (ESM, CJS, UMD)
- Vite uses Rollup under the hood for production

---

### esbuild — The speed king

**100x faster than traditional bundlers** due to Go implementation.

```bash
# Direct usage
npx esbuild src/index.tsx --bundle --outfile=dist/bundle.js --platform=browser --target=chrome90

# With watch
npx esbuild src/index.tsx --bundle --outfile=dist/bundle.js --watch
```

**Limitations:**
- No TypeScript type checking (only transpiles)
- No CSS Modules support
- Smaller plugin ecosystem
- **Best used as a transformer/transpiler** within other tools (Vite, Webpack 5 with esbuild-loader)

---

### Turbopack — Webpack's successor (Vercel)

- Written in **Rust**
- Designed to replace Webpack in Next.js
- Claims **700x faster** than Webpack in some benchmarks
- Incremental computation — only recomputes what changed
- Still in development/beta for general use

```bash
# Currently available via Next.js
next dev --turbopack
```

---

### SWC — Speedy Web Compiler

- Rust-based JavaScript/TypeScript compiler
- Drop-in replacement for Babel (20x faster)
- Used by Next.js as default compiler
- Not a full bundler — pairs with other tools

```bash
npm install --save-dev @swc/core @swc/cli

# Transform a file
npx swc src/index.ts -o dist/index.js
```

---

### Parcel — Zero-config bundler

```bash
# Zero configuration needed
npx parcel src/index.html
npx parcel build src/index.html
```

**Pros:**
- Zero config
- Good for quick prototypes
- Automatic transforms (Babel, TypeScript, SCSS)

**Cons:**
- Less control than Webpack
- Slower than Vite in dev
- Smaller ecosystem

---

### Comparison Table: All Bundlers

| Tool | Speed | Config | Use Case | Browser Support | Maturity |
|---|---|---|---|---|---|
| **Vite** | Fast (ESM dev) | Minimal | Apps + Libs | Modern | Growing |
| **Webpack 5** | Medium-Slow | Complex | Apps (any) | Any incl. IE11 | Mature |
| **Rollup** | Medium | Moderate | Libraries | Modern | Mature |
| **esbuild** | Fastest | Minimal | Transforms | Modern | Stable |
| **Turbopack** | Very Fast | Minimal | Apps (Next.js) | Modern | Beta |
| **Parcel** | Medium | Zero | Prototypes | Modern | Stable |
| **SWC** | Very Fast | Minimal | Transform only | Modern | Stable |

---

## Practical Scenarios

### How do you optimize a Vite production build?

```ts
// vite.config.ts — production optimization
export default defineConfig({
  build: {
    // Target modern browsers for smaller output
    target: 'es2020',
    
    // Minification options
    minify: 'esbuild',  // or 'terser' for smaller output
    
    // Source maps only for staging
    sourcemap: process.env.NODE_ENV === 'staging',
    
    // Chunk splitting
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules/react')) return 'react-vendor';
          if (id.includes('node_modules/@tanstack')) return 'query-vendor';
          if (id.includes('node_modules')) return 'vendor';
        },
        // Asset file naming with content hash
        assetFileNames: 'assets/[name]-[hash][extname]',
        chunkFileNames: 'assets/[name]-[hash].js',
        entryFileNames: 'assets/[name]-[hash].js',
      },
    },
    
    // Warn on large chunks
    chunkSizeWarningLimit: 500,
    
    // CSS code splitting
    cssCodeSplit: true,
    
    // Inline assets smaller than 4KB
    assetsInlineLimit: 4096,
  },
});
```

---

### How do you optimize a Webpack production build?

```js
// webpack.config.js — production optimizations
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
const TerserPlugin = require('terser-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

module.exports = {
  mode: 'production',
  
  output: {
    filename: '[name].[contenthash:8].js',
    chunkFilename: '[name].[contenthash:8].chunk.js',
    clean: true,
  },
  
  optimization: {
    minimizer: [
      new TerserPlugin({
        parallel: true,
        extractComments: false,
        terserOptions: {
          compress: {
            drop_console: true,
            drop_debugger: true,
          },
        },
      }),
      new CssMinimizerPlugin(),
    ],
    
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom|react-router-dom)[\\/]/,
          name: 'react',
          chunks: 'all',
          priority: 30,
        },
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          priority: 10,
        },
      },
    },
    
    runtimeChunk: 'single',
    moduleIds: 'deterministic',
  },
  
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash:8].css',
    }),
    process.env.ANALYZE && new BundleAnalyzerPlugin(),
  ].filter(Boolean),
  
  // Persistent filesystem cache
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename],
    },
  },
};
```

---

[← Back to README](./README.md)
