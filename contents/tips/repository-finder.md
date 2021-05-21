# 仓库


### [patch-package](https://github.com/ds300/patch-package)

> 使用`patch-package`修改第三方模块，及时解决第三方依赖包的`bug`

#### :man_mechanic: 用法:

##### 1. 安装

```bash
# npm
npm install patch-package --save-dev

# yarn
yarn add --dev patch-package postinstall-postinstall
```

##### 2. 创建补丁

> 直接在项目根目录下的 node_modules 文件夹中找到要修改依赖包的相关文件，然后回到根目录执行

```bash
# npm > 5.2
npx patch-package @my/package/@my/other-package

# yarn 
yarn patch-package @my/package/@my/other-package
```

##### 3. 修改源码

> 到`node_modules`修改对应源码


##### 4. 更新补丁

> 与创建补丁一致


##### 5. 打补丁

```bash
# npm
npx patch-package

# yarn
yarn patch-package
```

##### 6. 部署使用

> 在`package.json`的`scripts`中加入：

```text
"postinstall": "patch-package"
```

> 后续执行`npm install`或`yarn install`命令时，会自动为依赖包打补丁

```bash
# npm
npm instal

# yarn
yarn install
```

