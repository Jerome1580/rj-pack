# 简易版打包思路
1. new Compiler() 做两件事
    1. buildModule 打包各个模块
        1. 从入口文件出发，找到每个文件的依赖 和 对应代码，生成依赖的类似json对象
        2. 用 AST 替换 require 为 内部的 ' __webpack_require__ '，把文件的引用的后缀名补齐
    2. emitFile 发射文件，将打包后的文件写入 output 路径

# 执行
1. 入口出发，执行 compiler 的 run 方法
2. AST 转化
    1. babylon.parse 解析代码为 ast
    2. traverse 编译整个 ast 树
        1. CallExpression 是根据要修改的内容，比如要修改 require，在网站 https://astexplorer.net/ 找到对应的方法
    ![ast参考](https://tva1.sinaimg.cn/large/008i3skNly1gu0cvaw79aj61k50u0jv702.jpg)
        2. 参数 p 有个node节点，节点里有方法，可以找到对应的名字
    3. @babel/types 修改 ast 代码，
        - t.stringLiteral(moduleName)
    4. generator 根据新的 ast 树，生成对应的代码
    ```js
     // 解析源码
      parse(source, parentPath) { // AST解析语法树
        let ast = babylon.parse(source);
        let dependencies = [];// 依赖的数组
        traverse(ast, {
          CallExpression(p) { //  a() require()
            let node = p.node; // 对应的节点
            if (node.callee.name === 'require') {
              node.callee.name = '__webpack_require__';
              let moduleName = node.arguments[0].value; // 取到的就是模块的引用名字
              moduleName = moduleName + (path.extname(moduleName) ? '' : '.js');
              moduleName = './' + path.join(parentPath, moduleName);// 'src/a.js'
              dependencies.push(moduleName);
              node.arguments = [t.stringLiteral(moduleName)];
            }
          }
        });
        let sourceCode = generator(ast).code;
        return { sourceCode, dependencies }
      }
    ```
3. 根据新的代码，基于 ejs 模板，生成对应的打包后的代码
```js

// 模板代码（根据webpack自己打包后生成的文件，copy过来的）

(function (modules) {
  var installedModules = {};
  function __webpack_require__(moduleId) {
    if (installedModules[moduleId]) {
      return installedModules[moduleId].exports;
    }
    var module = installedModules[moduleId] = {
      i: moduleId,
      l: false,
      exports: {}
    };
    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
    module.l = true;
    return module.exports;
  }
  return __webpack_require__(__webpack_require__.s = "<%-entryId%>");
})
  ({
    <%for(let key in modules){%>
    "<%-key%>":
    (function (module, exports, __webpack_require__) {
      eval(`<%-modules[key]%>`);
    }),
    <%}%>
  });
```