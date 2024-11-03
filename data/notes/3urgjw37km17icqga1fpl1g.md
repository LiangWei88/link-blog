# 从零开始搭建React源码工程讲座讲义

一个培训机构面试的试讲讲义

录课的时候发现以下问题, 但是代码仓库是全的

用的话记得重头跑一遍来查漏补缺

> * babel 安装有问题
> * 创建文件路径有问题
> * react index.ts 演示代码代码没有贴

代码库:

https://gitee.com/imyyliang/react-clone

## 课程目标

本课程旨在带领学员从零开始搭建一个与React官方工程结构相近的源码工程。我们将使用Rollup进行编译打包，Jest进行测试，pnpm进行包管理，并搭建一个monorepo结构的多包工程。课程将涵盖以下内容：

1. 项目初始化与目录结构搭建
2. 配置TypeScript与ESLint
3. 使用Rollup进行打包
4. 使用Jest进行单元测试
5. 搭建monorepo结构并处理模块间依赖
6. 创建示例页面并实现热更新

## 课程大纲

### 1. 项目初始化与目录结构搭建

#### 1.1 创建项目

首先，我们创建一个新的项目目录并初始化项目。

```bash
mkdir react-clone
cd react-clone
pnpm init
git init
git add .
git commit -m '初始化'
```

#### 1.2 准备文件夹和文件

接下来，我们创建项目的目录结构，并为每个模块准备必要的文件。

```bash
mkdir packages scripts
touch eslint.config.js

cd packages
mkdir react react-dom scheduler
mkdir react/src
mkdir react/src/__test__
touch react/src/index.ts react/src/__test__/index.test.ts
# 上面两个文件也复制到 react-dom 和 scheduler

cd ../scripts
mkdir rollup jest babel
touch jest\jest.config.js babel\babel.config.js rollup\rollup.config.js

cd ..
```

### 2. 配置TypeScript与ESLint

#### 2.1 配置TypeScript

在根目录下创建`tsconfig.json`文件，并配置TypeScript编译选项。

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "strict": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "node",
    "isolatedModules": true,
    "noEmit": true,
    "importHelpers": true,
    "lib": ["dom", "ESNext"]
  },
  "include": ["packages/**/*", "scripts/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

#### 2.2 配置ESLint

在根目录下创建`eslint.config.js`文件，并配置ESLint规则。

```js
const path = require('path');

module.exports = [
  {
    files: ['packages/**/*.ts', 'scripts/**/*.ts'],
    ignores: ['**/node_modules/**/*', '**/dist/**/*', '**/build/**/*'],
    languageOptions: {
      parserOptions: {
        ecmaVersion: 'latest',
        sourceType: 'module',
        project: path.resolve(__dirname, './tsconfig.json'),
      },
      parser: require('@typescript-eslint/parser'), // 确保使用 TypeScript 解析器
    },
    plugins: {
      '@typescript-eslint': require('@typescript-eslint/eslint-plugin'),
    },
    rules: {
      '@typescript-eslint/explicit-module-boundary-types': 'off',
      '@typescript-eslint/no-explicit-any': 'off',
      'no-console': 'warn',
      '@typescript-eslint/no-unused-vars': ['error'],
    },
    settings: {
      'import/resolver': {
        typescript: {
          project: path.resolve(__dirname, './tsconfig.json'),
        },
      },
    },
  },
];
```

#### 2.3 安装依赖

安装TypeScript和ESLint相关依赖。

```bash
pnpm add -D -w eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin typescript
```

#### 2.4 测试ESLint配置

在任何一个`src/index.ts`文件中写入`console.log`，然后运行`pnpm lint`，应该会看到警告信息。

### 3. 使用Rollup进行打包

#### 3.1 安装Rollup及相关插件

安装Rollup及其插件。

```bash
pnpm add -D -w rollup rollup-plugin-commonjs rollup-plugin-node-resolve rollup-plugin-typescript
```

#### 3.2 配置Rollup

在`scripts/rollup/rollup.config.js`中配置Rollup。

```js
const resolve = require('rollup-plugin-node-resolve');
const commonjs = require('rollup-plugin-commonjs');
const typescript = require('rollup-plugin-typescript');
const path = require('path');
const fs = require('fs');

// 获取所有模块的路径
const packagesDir = path.resolve(__dirname, '../../packages');
const packages = fs.readdirSync(packagesDir).filter(pkg => {
  const stat = fs.statSync(path.join(packagesDir, pkg));
  return stat.isDirectory();
});

// 构建每个模块的输入和输出配置
const inputConfigs = packages.map(pkg => ({
  input: path.join(packagesDir, pkg, 'src', 'index.ts'), // 每个模块的入口
  output: [
    {
      file: path.join(packagesDir, pkg, 'dist', 'index.umd.js'),
      format: 'umd',
      name: pkg.charAt(0).toUpperCase() + pkg.slice(1), // 生成全局名称
    },
    {
      file: path.join(packagesDir, pkg, 'dist', 'index.esm.js'),
      format: 'esm',
    },
    {
      file: path.join(packagesDir, pkg, 'dist', 'index.cjs.js'),
      format: 'cjs',
    },
  ],
  plugins: [
    resolve(),
    commonjs(),
    typescript()
  ],
}));

module.exports = inputConfigs;
```

#### 3.3 配置`package.json`

在`package.json`中添加`build`脚本。

```json
  "scripts": {
    "lint": "eslint",
    "build": "rollup -c scripts/rollup/rollup.config.js"
  },
```

#### 3.4 测试Rollup配置

运行`pnpm build`命令，应该会生成各个模块下方的`dist/index.xxx.js`文件。

### 4. 使用Jest进行单元测试

#### 4.1 安装Jest及相关依赖

安装Jest及其相关依赖。

```bash
pnpm add -D -w jest jest-environment-jsdom @types/jest babel-jest
```

#### 4.2 配置Jest

在`scripts/jest/jest.config.js`中配置Jest。

```js
module.exports = {
  transform: {
    '^.+\\.ts$': ['babel-jest', {configFile: require.resolve('../babel/babel.config.js')}],
  },
  rootDir: process.cwd(),
  roots: ['<rootDir>/packages', '<rootDir>/scripts'],
  testEnvironment: 'jsdom'
};
```

#### 4.3 配置Babel

在`scripts/babel/babel.config.js`中配置Babel。

```js
// scripts/babel/babel.config.js
module.exports = {
  presets: [
    ['@babel/preset-env', { targets: { node: 'current' } }],
    '@babel/preset-typescript',
  ]
};
```

#### 4.4 编写测试用例

在`packages/react/src/__test__/index.test.ts`中编写测试用例。

```js
import { React } from '..';

test('创建一个div元素', () => {
  const element1 = React.createElement('div', {}, '这是一个div元素');
  expect(element1.type).toBe('div');
  expect(element1.props).toEqual({});
  expect(element1.children[0]).toBe('这是一个div元素');
});

test('创建一个p元素，带有属性class="myClass"', () => {
  const element2 = React.createElement('p', { class: 'myClass' }, '这是一个p元素');
  expect(element2.type).toBe('p');
  expect(element2.props).toEqual({ class: 'myClass' });
  expect(element2.children[0]).toBe('这是一个p元素');
});

test('创建一个div元素，包含一个p元素', () => {
  const element3 = React.createElement('div', {}, React.createElement('p', {}, '这是一个p元素'));
  expect(element3.type).toBe('div');
  expect(element3.props).toEqual({});
  expect(element3.children[0].type).toBe('p');
  expect(element3.children[0].props).toEqual({});
  expect(element3.children[0].children[0]).toBe('这是一个p元素');
});
```

#### 4.5 配置`package.json`

在`package.json`中添加`test`脚本。

```json
  "scripts": {
    "lint": "eslint",
    "build": "rollup -c scripts/rollup/rollup.config.js",
    "test": "jest -c scripts/jest/jest.config.js"
  },
```

#### 4.6 测试Jest配置

运行`pnpm test`命令，应该会运行所有测试用例并输出结果。

### 5. 搭建monorepo结构并处理模块间依赖

#### 5.1 初始化monorepo

在每个模块目录下执行`pnpm init`，并修改生成的`package.json`文件，将`main`字段改为`src/index.ts`。

```bash
cd packages/react
pnpm init
# 修改 package.json 中的 main 字段为 src/index.ts

cd ../react-dom
pnpm init
# 修改 package.json 中的 main 字段为 src/index.ts

cd ../scheduler
pnpm init
# 修改 package.json 中的 main 字段为 src/index.ts
```

#### 5.2 配置工作区

在根目录下创建`pnpm-workspace.yaml`文件，配置工作区。

```yaml
packages:
  - 'packages/*'
```

#### 5.3 处理模块间依赖

在`react`模块的`package.json`中添加对`scheduler`模块的依赖。

```json
  "dependencies": {
    "scheduler": "workspace:*"
  }
```

在`react`模块目录下运行`pnpm install`。

#### 5.4 测试monorepo配置

在`react`模块中使用`scheduler`模块，并编写测试用例。

```js
import { Scheduler } from 'scheduler'
export class React {
  static createElement(type: string, props: any, ...children: any[]) {
    return { type, props, children };
  }

  static scheduleReconciliation() {
    Scheduler.scheduleWork(() => {
      console.log('Reconciling...');
    });
  }
}
```

```js
import { React } from '..';
import { Scheduler } from 'scheduler';

jest.mock('scheduler', () => ({
  Scheduler: {
    scheduleWork: jest.fn()
  }
}));

describe('React 类', () => {
  describe('createElement 方法', () => {
    test('创建一个div元素', () => {
      const element1 = React.createElement('div', {}, '这是一个div元素');
      expect(element1.type).toBe('div');
      expect(element1.props).toEqual({});
      expect(element1.children[0]).toBe('这是一个div元素');
    });

    test('创建一个p元素，带有属性class="myClass"', () => {
      const element2 = React.createElement('p', { class: 'myClass' }, '这是一个p元素');
      expect(element2.type).toBe('p');
      expect(element2.props).toEqual({ class: 'myClass' });
      expect(element2.children[0]).toBe('这是一个p元素');
    });

    test('创建一个div元素，包含一个p元素', () => {
      const element3 = React.createElement('div', {}, React.createElement('p', {}, '这是一个p元素'));
      expect(element3.type).toBe('div');
      expect(element3.props).toEqual({});
      expect(element3.children[0].type).toBe('p');
      expect(element3.children[0].props).toEqual({});
      expect(element3.children[0].children[0]).toBe('这是一个p元素');
    });
  });

  describe('scheduleReconciliation 方法', () => {
    test('应该调用 Scheduler.scheduleWork', () => {
      React.scheduleReconciliation();
      expect(Scheduler.scheduleWork).toHaveBeenCalled();
    });
  });
});
```

运行`pnpm test`进行测试。

### 6. 创建示例页面并实现热更新

#### 6.1 准备模块文件

创建`example`模块，并准备必要的文件。

```bash
mkdir packages/example
cd packages/example
touch index.html index.ts
pnpm init
```

#### 6.2 配置`example`模块

在`example/index.html`中编写HTML文件。

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>React Example</title>
</head>
<body>
  <div id="root"></div>
  <script src="dist/bundle.js"></script>
</body>
</html>
```

在`example/index.ts`中编写TypeScript代码。

```js
import { React } from 'react';
import { ReactDOM } from 'react-dom';

const element = React.createElement(
  'div',
  null,
  React.createElement('h1', null, 'Hello, World!'),
  React.createElement('p', null, 'This is a simple React component.')
);

ReactDOM.render(element, document.getElementById('root')!);
```

在`example/package.json`中添加依赖。

```json
  "dependencies": {
    "react": "workspace:*",
    "react-dom": "workspace:*"
  }
```

在`example`模块目录下运行`pnpm install`。

#### 6.3 配置Rollup

在`scripts/rollup/rollup.config.js`中配置Rollup，支持热更新。

```js
const resolve = require('rollup-plugin-node-resolve');
const commonjs = require('rollup-plugin-commonjs');
const typescript = require('rollup-plugin-typescript');
const serve = require('rollup-plugin-serve');
const livereload = require('rollup-plugin-livereload');
const path = require('path');
const fs = require('fs');
const { execSync } = require('child_process');

const packagesDir = path.resolve(__dirname, '../../packages');

// 获取命令行参数
const argv = process.argv.slice(2);
const isWatchMode = argv.includes('-w') || argv.includes('--watch');

// 获取所有模块的路径
const packages = fs.readdirSync(packagesDir).filter(pkg => {
  const stat = fs.statSync(path.join(packagesDir, pkg));
  return stat.isDirectory() && pkg !== 'example';
});

// 构建每个模块的输入和输出配置
const buildPackageConfig = (pkg) => ({
  input: path.join(packagesDir, pkg, 'src', 'index.ts'),
  output: [
    {
      file: path.join(packagesDir, pkg, 'dist', 'index.umd.js'),
      format: 'umd',
      name: pkg.charAt(0).toUpperCase() + pkg.slice(1),
    },
    {
      file: path.join(packagesDir, pkg, 'dist', 'index.esm.js'),
      format: 'esm',
    },
    {
      file: path.join(packagesDir, pkg, 'dist', 'index.cjs.js'),
      format: 'cjs',
    },
  ],
  plugins: [
    resolve(),
    commonjs(),
    typescript(),
  ],
});

// 构建 example 页面的打包配置
const buildExampleConfig = () => ({
  input: path.join(packagesDir, 'example', 'index.ts'),
  output: {
    file: path.join(packagesDir, 'example', 'dist', 'bundle.js'),
    format: 'iife',
    sourcemap: true,
  },
  plugins: [
    resolve(),
    commonjs(),
    typescript(),
    serve({
      contentBase: [path.join(packagesDir, 'example')],
      port: 3000,
    }),
    livereload({
      watch: path.join(packagesDir, 'example', 'dist'),
    }),
  ],
});

// 根据模式选择配置
const inputConfigs = isWatchMode
  ? [buildExampleConfig()]
  : packages.map(buildPackageConfig);

// 在所有模块打包完成后安装依赖
inputConfigs.forEach(config => {
  config.plugins.push({
    name: 'pnpm-install',
    buildEnd() {
      console.log(`Running pnpm install for ${config.input}`);
      execSync('pnpm install', { cwd: path.dirname(config.input) });
    },
  });
});

// 导出所有配置
module.exports = inputConfigs;
```

#### 6.4 配置`package.json`

在`package.json`中添加`start`脚本。

```json
  "scripts": {
    "lint": "eslint",
    "build": "rollup -c scripts/rollup/rollup.config.js",
    "test": "jest -c scripts/jest/jest.config.js",
    "start": "rollup -c scripts/rollup/rollup.config.js -w"
  },
```

#### 6.5 运行测试

在根目录下运行`pnpm start`，然后打开`http://localhost:3000/`，应该能看到页面，同时修改`example/index.ts`可以测试热更新功能。

## 总结

通过本课程，我们成功搭建了一个与React官方工程结构相近的源码工程。我们使用了Rollup进行打包，Jest进行单元测试，pnpm进行包管理，并搭建了一个monorepo结构的多包工程。我们还创建了一个示例页面并实现了热更新功能。希望本课程能帮助你更好地理解React源码工程的搭建过程。