# test

This template should help get you started developing with Vue 3 in Vite.

## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur).

## Type Support for `.vue` Imports in TS

TypeScript cannot handle type information for `.vue` imports by default, so we replace the `tsc` CLI with `vue-tsc` for type checking. In editors, we need [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) to make the TypeScript language service aware of `.vue` types.

## Customize configuration

See [Vite Configuration Reference](https://vite.dev/config/).

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Type-Check, Compile and Minify for Production

```sh
npm run build
```

### Run Unit Tests with [Vitest](https://vitest.dev/)

```sh
npm run test:unit
```

### Lint with [ESLint](https://eslint.org/)

```sh
npm run lint
```

### Git Initial
```
git init # 初始化 git
git add . # 將所有檔案加入 git 索引
git commit -m "first commit" # 加上 commit 訊息
git branch -M main # 將分支名稱改成 main
git remote add origin git@github.com:xxx.git # 新增遠端儲存庫路徑
git push -u origin main # 將本地端的 main 分支推送到遠端儲存庫，並且設定為預設分支，下次就可以直接使用 git push


npm init -y # 初始化 npm
npm install gh-pages # 安裝 gh-pages 套件

add to package.json
"deploy": "gh-pages -d dist" 
"deploy": "vite build && gh-pages -d dist"

```