### webpack-dev-server在我们的entry中添加的hot模块内容

看看下面的方法你就知道了，在hot模式下，我们的entry最后都会被添加两个文件：

```js
module.exports = function addDevServerEntrypoints(webpackOptions, devServerOptions) {
    if(devServerOptions.inline !== false) {
        //表示是inline模式而不是iframe模式
        const domain = createDomain(devServerOptions);
        const devClient = [`${require.resolve("../../client/")}?${domain}`];
        //客户端内容
        if(devServerOptions.hotOnly)
            devClient.push("webpack/hot/only-dev-server");
        else if(devServerOptions.hot)
            devClient.push("webpack/hot/dev-server");
        //配置了不同的webpack而文件到客户端文件中
        [].concat(webpackOptions).forEach(function(wpOpt) {
            if(typeof wpOpt.entry === "object" && !Array.isArray(wpOpt.entry)) {
                /*
                  entry:{
                    index:'./index.js',    => entry:[]
                    index1:'./index1.js'
                  }
                 */
                Object.keys(wpOpt.entry).forEach(function(key) {
                    wpOpt.entry[key] = devClient.concat(wpOpt.entry[key]);
                });
                //添加我们自己的入口文件
            } else if(typeof wpOpt.entry === "function") {
                wpOpt.entry = wpOpt.entry(devClient);
                //如果entry是一个函数那么我们把devClient数组传入
            } else {
                wpOpt.entry = devClient.concat(wpOpt.entry);
                //数组直接传入
            }
        });
    }
};
```

（1）首先看看"webpack/hot/only-dev-server"的文件内容：

```js
if(module.hot) {
    var lastHash;
    //___webpack_hash__
    //Access to the hash of the compilation.
    //Only available with the HotModuleReplacementPlugin or the ExtendedAPIPlugin
    var upToDate = function upToDate() {
        return lastHash.indexOf(__webpack_hash__) >= 0;
        //如果两个hash相同那么表示没有更新
    };
    //检查更新
    var check = function check() {
        //Check all currently loaded modules for updates and apply updates if found.
        module.hot.check().then(function(updatedModules) {
            //没有更新的模块直接返回
            if(!updatedModules) {
                console.warn("[HMR] Cannot find update. Need to do a full reload!");
                console.warn("[HMR] (Probably because of restarting the webpack-dev-server)");
                return;
            }
            //apply方法:If status() != "ready" it throws an error.
            //开始更新
            return module.hot.apply({
                ignoreUnaccepted: true,
                ignoreDeclined: true,
                ignoreErrored: true,
                onUnaccepted: function(data) {
                    console.warn("Ignored an update to unaccepted module " + data.chain.join(" -> "));
                },
                onDeclined: function(data) {
                    console.warn("Ignored an update to declined module " + data.chain.join(" -> "));
                },
                onErrored: function(data) {
                    console.warn("Ignored an error while updating module " + data.moduleId + " (" + data.type + ")");
                }
             //renewedModules表示哪些模块已经更新了
            }).then(function(renewedModules) {
                if(!upToDate()) {
                    check();
                }
                //更新的模块updatedModules，renewedModules表示哪些模块已经更新了
                require("./log-apply-result")(updatedModules, renewedModules);
                if(upToDate()) {
                    console.log("[HMR] App is up to date.");
                }
            });
        }).catch(function(err) {
            var status = module.hot.status();
            if(["abort", "fail"].indexOf(status) >= 0) {
                console.warn("[HMR] Cannot check for update. Need to do a full reload!");
                console.warn("[HMR] " + err.stack || err.message);
            } else {
                console.warn("[HMR] Update check failed: " + err.stack || err.message);
            }
        });
    };
    var hotEmitter = require("./emitter");
    //emitter模块内容，也就是导出一个events实例
    /*
    var EventEmitter = require("events");
    module.exports = new EventEmitter();
     */
    hotEmitter.on("webpackHotUpdate", function(currentHash) {
        lastHash = currentHash;
        //表示本次更新后得到的hash值
        if(!upToDate()) {
            //有更新
            var status = module.hot.status();
            if(status === "idle") {
                console.log("[HMR] Checking for updates on the server...");
                check();
            } else if(["abort", "fail"].indexOf(status) >= 0) {
                console.warn("[HMR] Cannot apply update as a previous update " + status + "ed. Need to do a full reload!");
            }
        }
    });
    console.log("[HMR] Waiting for update signal from WDS...");
} else {
    throw new Error("[HMR] Hot Module Replacement is disabled.");
}

```

./log-apply-result模块内容如下：

```js
module.exports = function(updatedModules, renewedModules) {
    //renewedModules表示哪些模块被更新了，剩余的模块表示，哪些模块由于 ignoreDeclined，ignoreUnaccepted配置没有更新
    var unacceptedModules = updatedModules.filter(function(moduleId) {
        return renewedModules && renewedModules.indexOf(moduleId) < 0;
    });
    //哪些模块无法HMR，打印log
    if(unacceptedModules.length > 0) {
        console.warn("[HMR] The following modules couldn't be hot updated: (They would need a full reload!)");
        unacceptedModules.forEach(function(moduleId) {
            console.warn("[HMR]  - " + moduleId);
        });
    }
    //没有模块更新，表示模块是最新的
    if(!renewedModules || renewedModules.length === 0) {
        console.log("[HMR] Nothing hot updated.");
    } else {
        console.log("[HMR] Updated modules:");
        //更新的模块
        renewedModules.forEach(function(moduleId) {
            console.log("[HMR]  - " + moduleId);
        });
        //每一个moduleId都是数字那么建议使用NamedModulesPlugin
        var numberIds = renewedModules.every(function(moduleId) {
            return typeof moduleId === "number";
        });
        if(numberIds)
            console.log("[HMR] Consider using the NamedModulesPlugin for module names.");
    }
};
```

所以"webpack/hot/only-dev-server"的文件内容就是检查哪些模块更新了(通过webpackHotUpdate事件完成)，其中哪些模块更新成功，而哪些模块由于某种原因没有更新成功。其中没有更新的原因可能是如下的:

<pre>
    ignoreUnaccepted
    ignoreDecline
    ignoreErrored
</pre>

至于模块什么时候接受到需要更新是和webpack的打包过程有关的，这里也给出触发更新的时机:

```js
 ok: function() {
        sendMsg("Ok");
        if(useWarningOverlay || useErrorOverlay) overlay.clear();
        if(initial) return initial = false;
        reloadApp();
    },
    warnings: function(warnings) {
        log("info", "[WDS] Warnings while compiling.");
        var strippedWarnings = warnings.map(function(warning) {
            return stripAnsi(warning);
        });
        sendMsg("Warnings", strippedWarnings);
        for(var i = 0; i < strippedWarnings.length; i++)
            console.warn(strippedWarnings[i]);
        if(useWarningOverlay) overlay.showMessage(warnings);

        if(initial) return initial = false;
        reloadApp();
    },
function reloadApp() {
    //如果开启了HMR模式
    if(hot) {
        log("info", "[WDS] App hot update...");
        var hotEmitter = require("webpack/hot/emitter");
        hotEmitter.emit("webpackHotUpdate", currentHash);
        //重新启动webpack/hot/emitter，同时设置当前hash
        if(typeof self !== "undefined" && self.window) {
            // broadcast update to window
            self.postMessage("webpackHotUpdate" + currentHash, "*");
        }
    } else {
       //如果不是Hotupdate那么我们直接reload我们的window就可以了
        log("info", "[WDS] App updated. Reloading...");
        self.location.reload();
    }
}

```

也就是说当客户端接受到服务器端发送的ok和warning信息的时候，同时支持HMR的情况下就会要求检查更新，同时发送过来的还有服务器端本次编译的hash值。我们继续深入一步，看看服务器什么时候发送'ok'和'warning'消息：

```js
Server.prototype._sendStats = function(sockets, stats, force) {
    if(!force &&
        stats &&
        (!stats.errors || stats.errors.length === 0) &&
        stats.assets &&
        stats.assets.every(function(asset) {
            return !asset.emitted;
            //每一个asset都是没有emitted属性，表示没有发生变化。如果发生变化那么这个assets肯定有emitted属性
        })
    )
        return this.sockWrite(sockets, "still-ok");
    this.sockWrite(sockets, "hash", stats.hash);
    //设置hash
    if(stats.errors.length > 0)
        this.sockWrite(sockets, "errors", stats.errors);
    else if(stats.warnings.length > 0)
        this.sockWrite(sockets, "warnings", stats.warnings);
    else
        this.sockWrite(sockets, "ok");
}
```

也就是说更新是通过上面这个方法完成的，我们看看上面这个方法什么时候调用就可以了：

```js
compiler.plugin("done", function(stats) {
        this._sendStats(this.sockets, stats.toJson(clientStats));
        this._stats = stats;
    }.bind(this));
```

是不是豁然开朗了，也就是每次compiler的'done'钩子函数被调用的时候就会要求客户端去检查模块更新，进而完成HMR基本功能！

（2）再来看看webpack/hot/dev-server

```js
if(module.hot) {
    var lastHash;
    //__webpack_hash__是每次编译的hash值是全局的
    //Only available with the HotModuleReplacementPlugin or the ExtendedAPIPlugin
    var upToDate = function upToDate() {
        return lastHash.indexOf(__webpack_hash__) >= 0;
    };
    var check = function check() {
   // check([autoApply], callback: (err: Error, outdatedModules: Module[]) => void
   // If autoApply is truthy the callback will be called with all modules that were disposed. apply() is automatically called with autoApply as options parameter.(传入哪些代码已经被更新的模块)
   //If autoApply is not set the callback will be called with all modules that will be disposed on apply(). （不是true那么传入的是哪些需要被apply处理的模块）
        module.hot.check(true).then(function(updatedModules) {
            //检查所有要更新的模块，如果没有模块要更新那么回调函数就是null
            if(!updatedModules) {
                console.warn("[HMR] Cannot find update. Need to do a full reload!");
                console.warn("[HMR] (Probably because of restarting the webpack-dev-server)");
                window.location.reload();
                return;
            }
            //如果还有更新
            if(!upToDate()) {
                check();
            }
            require("./log-apply-result")(updatedModules, updatedModules);
            //已经被更新的模块都是updatedModules
            if(upToDate()) {
                console.log("[HMR] App is up to date.");
            }

        }).catch(function(err) {
            var status = module.hot.status();
            //如果报错直接全局reload
            if(["abort", "fail"].indexOf(status) >= 0) {
                console.warn("[HMR] Cannot apply update. Need to do a full reload!");
                console.warn("[HMR] " + err.stack || err.message);
                window.location.reload();
            } else {
                console.warn("[HMR] Update failed: " + err.stack || err.message);
            }
        });
    };
    var hotEmitter = require("./emitter");
    //获取MyEmitter对象
    hotEmitter.on("webpackHotUpdate", function(currentHash) {
        lastHash = currentHash;
        if(!upToDate() && module.hot.status() === "idle") {
            //调用module.hot.status方法获取状态
            console.log("[HMR] Checking for updates on the server...");
            check();
        }
    });
    console.log("[HMR] Waiting for update signal from WDS...");
} else {
    throw new Error("[HMR] Hot Module Replacement is disabled.");
}
```

也就是说webpack/hot/dev-server相较于前面在入口文件中添加的"webpack/hot/only-dev-server"来说，区别在于后者传入的是哪些已经被更新的模块，也就是已经被自己模块本身dispose处理了。如下：

```js
if (module.hot) {
    module.hot.accept();
    // dispose handler
    module.hot.dispose(() => {
        window.clearInterval(intervalId);
    });
}
```

(3)如果你注意到上面其实我们还添加了一个client/index.js，这个客户端代码只是添加了我们的客户端的socket.js代码，这时候我们客户端就可以获取到服务器端发送到的socket命令

```js
var onSocketMsg = {
    //设置hot为true
    hot: function() {
        hot = true;
        log("info", "[WDS] Hot Module Replacement enabled.");
    },
    //打印invalid
    invalid: function() {
        log("info", "[WDS] App updated. Recompiling...");
        sendMsg("Invalid");
    },
    //设置hash
    hash: function(hash) {
        currentHash = hash;
    },
    //继续可用
    "still-ok": function() {
        log("info", "[WDS] Nothing changed.")
        if(useWarningOverlay || useErrorOverlay) overlay.clear();
        sendMsg("StillOk");
    },
    //设置log级别
    "log-level": function(level) {
        logLevel = level;
    },
    /*
    Shows a full-screen overlay in the browser when there are compiler errors or warnings.
    Disabled by default. If you want to show only compiler errors:
    overlay: true
    If you want to show warnings as well as errors:
    overlay: {
      warnings: true,
      errors: true
    }
     */
    "overlay": function(overlay) {
        if(typeof document !== "undefined") {
            if(typeof(overlay) === "boolean") {
                useWarningOverlay = overlay;
                useErrorOverlay = overlay;
            } else if(overlay) {
                useWarningOverlay = overlay.warnings;
                useErrorOverlay = overlay.errors;
            }
        }
    },
    //ok
    ok: function() {
        sendMsg("Ok");
        if(useWarningOverlay || useErrorOverlay) overlay.clear();
        if(initial) return initial = false;
        reloadApp();
    },
    //客户端检测到服务器端有更新,通过chokidar检测到文件的变化
    "content-changed": function() {
        log("info", "[WDS] Content base changed. Reloading...")
        self.location.reload();
    },
    warnings: function(warnings) {
        log("info", "[WDS] Warnings while compiling.");
        var strippedWarnings = warnings.map(function(warning) {
            return stripAnsi(warning);
        });
        sendMsg("Warnings", strippedWarnings);
        for(var i = 0; i < strippedWarnings.length; i++)
            console.warn(strippedWarnings[i]);
        if(useWarningOverlay) overlay.showMessage(warnings);

        if(initial) return initial = false;
        reloadApp();
    },
    errors: function(errors) {
        log("info", "[WDS] Errors while compiling. Reload prevented.");
        var strippedErrors = errors.map(function(error) {
            return stripAnsi(error);
        });
        sendMsg("Errors", strippedErrors);
        for(var i = 0; i < strippedErrors.length; i++)
            console.error(strippedErrors[i]);
        if(useErrorOverlay) overlay.showMessage(errors);
    },
    //发送消息close
    close: function() {
        log("error", "[WDS] Disconnected!");
        sendMsg("Close");
    }
};
socket(socketUrl, onSocketMsg);
```

### module.hot等相关方法什么时候被调用

其实通过上面的分析，我们肯定有一点疑问就是，我们的这些module.hot等方法是在什么时候调用的，其实看看[HotModuleReplacementPlugin](https://github.com/liangklfang/webpack/blob/master/lib/HotModuleReplacementPlugin.js#L48)就明白了，下面贴出一部分代码：

```js
     parser.plugin("call module.hot.accept", function(expr) {
                if(!this.state.compilation.hotUpdateChunkTemplate) return false;
                if(expr.arguments.length >= 1) {
                    var arg = this.evaluateExpression(expr.arguments[0]);
                    var params = [],
                        requests = [];
                    if(arg.isString()) {
                        params = [arg];
                    } else if(arg.isArray()) {
                        params = arg.items.filter(function(param) {
                            return param.isString();
                        });
                    }
                    if(params.length > 0) {
                        params.forEach(function(param, idx) {
                            var request = param.string;
                            var dep = new ModuleHotAcceptDependency(request, param.range);
                            dep.optional = true;
                            dep.loc = Object.create(expr.loc);
                            dep.loc.index = idx;
                            this.state.module.addDependency(dep);
                            requests.push(request);
                        }.bind(this));
                        if(expr.arguments.length > 1)
                            this.applyPluginsBailResult("hot accept callback", expr.arguments[1], requests);
                        else
                            this.applyPluginsBailResult("hot accept without callback", expr, requests);
                    }
                }
            });
            parser.plugin("call module.hot.decline", function(expr) {
                if(!this.state.compilation.hotUpdateChunkTemplate) return false;
                if(expr.arguments.length === 1) {
                    var arg = this.evaluateExpression(expr.arguments[0]);
                    var params = [];
                    if(arg.isString()) {
                        params = [arg];
                    } else if(arg.isArray()) {
                        params = arg.items.filter(function(param) {
                            return param.isString();
                        });
                    }
                    params.forEach(function(param, idx) {
                        var dep = new ModuleHotDeclineDependency(param.string, param.range);
                        dep.optional = true;
                        dep.loc = Object.create(expr.loc);
                        dep.loc.index = idx;
                        this.state.module.addDependency(dep);
                    }.bind(this));
                }
            });
            parser.plugin("expression module.hot", function() {
                return true;
            });
        });
    });
```

也就是我们关注的这些module.hot.decline方法都是在Parser上封装的！

### 如何写出支持HMR的代码

这里就是一个例子,你也可以查看[这个仓库](https://github.com/liangklfangl/wcf),然后克隆下来，执行"node ./bin/wcf --dev"命令，你就会发现访问localhost:8080的时候代码是可以支持HMR(你可以修改test目录下的所有的文件)，而不会出现页面刷新的情况。

```js
import * as dom from './dom';
import * as time from './time';
import pulse from './pulse';
require('./styles.scss');
const UPDATE_INTERVAL = 1000; // milliseconds
const intervalId = window.setInterval(() => {
    dom.writeTextToElement('upTime', time.getElapsedSeconds() + ' seconds');
    dom.writeTextToElement('lastPulse', pulse());
}, UPDATE_INTERVAL);
// Activate Webpack HMR
if (module.hot) {
    module.hot.accept();
    // dispose handler
    module.hot.dispose(() => {
        window.clearInterval(intervalId);
    });
}
```

####其中accept函数签名如下:

```js
accept(dependencies: string[], callback: (updatedDependencies) => void) => void
accept(dependency: string, callback: () => void) => void
//直接接受当前模块某一个依赖模块的HMR
```

此时表示，我们这个模块支持HMR，任何其依赖的模块变化都会被捕捉到。当依赖的模块更新后回调函数被调用。当然，如果是下面这种方式：

```js
accept([errHandler]) => void
```

那么表示我们接受当前模块所有依赖的模块的代码更新，而且这种更新不会冒泡到父级中去。这当我们模块没有导出任何东西的情况下有用(比如entry)。

#### 其中decline函数

上面的例子中我们的dom.js是如下方式写的：

```js
import $ from 'jquery';
export function writeTextToElement(id, text) {
    $('#' + id).text(text);
}
if (module.hot) {
    module.hot.decline('jquery');//不接受jquery更新
}
```

其中decline方法签名如下:

```js
decline(dependencies: string[]) => void
decline(dependency: string) => void
```

这表明我们不会接受特定模块的更新，如果该模块更新了，那么更新失败同时失败代码为"decline"。而上面的代码表明我们不会接受jquery模块的更新。当前也可以是如下模式：

```js
decline() => void
```

这表明我们当前的模块是不会更新的，也就是不会HMR。如果更新了那么错误代码为"decline";

#### 其中dispose函数

函数签名如下:

```js
dispose(callback: (data: object) => void) => void
addDisposeHandler(callback: (data: object) => void) => void
```

这表示我们会添加一个一次性的处理函数，这个函数在当前模块更新后会被调用。此时，你需要移除或者销毁一些持久的资源，如果你想将当前的状态信息转移到更新后的模块中，此时可以添加到data对象中，以后可以通过module.hot.data访问。如下面的例子用于保存指定模块实例化的时间，从而防止模块更新后数据丢失(刷新后还是会丢失的)。

```js
let moduleStartTime = getCurrentSeconds();
function getCurrentSeconds() {
    return Math.round(new Date().getTime() / 1000);
    // return new Date().getTime() / 1000;
}
export function getElapsedSeconds() {
    return getCurrentSeconds() - moduleStartTime;
}
// Activate Webpack HMR
if (module.hot) {
    const data = module.hot.data || {};
    // Update our moduleStartTime if we are in the process of reloading
    if (data.moduleStartTime)
        moduleStartTime = data.moduleStartTime;
    // dispose handler to pass our moduleStart time to the next version of our module
    // 首次进入我们把当前时间保存到moduleStartTime中以后就可以直接访问
    module.hot.dispose((data) => {
        data.moduleStartTime = moduleStartTime;
    });
}
```

### hotUpdateChunkFilename vs hotUpdateMainFilename

当你修改了test目录下的文件的时候，比如修改了scss文件，此时你会发现在页面中多出了一个script元素，内容如下:

```html
<script type="text/javascript" charset="utf-8" src="0.188304c98f697ecd01b3.hot-update.js"></script>
```

其中内容是：

```js
webpackHotUpdate(0,{
/***/ 15:
/***/ (function(module, exports, __webpack_require__) {
exports = module.exports = __webpack_require__(46)();
// imports
// module
exports.push([module.i, "html {\n  border: 1px solid yellow;\n  background-color: pink; }\n\nbody {\n  background-color: lightgray;\n  color: black; }\n  body div {\n    font-weight: bold; }\n    body div span {\n      font-weight: normal; }\n", ""]);
// exports

/***/ })
})
//# sourceMappingURL=0.188304c98f697ecd01b3.hot-update.js.map
```

从内容你也可以看出，只是将我们修改的模块push到exports对象中！而hotUpdateChunkFilename就是为了让你能够执行script的src中的值的！而同样的hotUpdateMainFilename是一个json文件用于指定哪些模块发生了变化，在output目录下。

### webpack和webpack-dev-server关系

webpack首先打包成文件放在具体的目录下，并通过publicPath配置成了虚拟路径，而当通过URL访问服务器的时候就会从req.url寻找了具体的输出文件，最后得到这个输出文件并原样发送到客户端。看下面的方法你就明白了：

```js
var pathJoin = require("./PathJoin");
var urlParse = require("url").parse;
function getFilenameFromUrl(publicPath, outputPath, url) {
    var filename;
    // localPrefix is the folder our bundle should be in
    // 第二个参数如果为false那么查询字符串不会被decode或者解析
    // 第三个参数为true,那么//foo/bar被解析为{host: 'foo', pathname: '/bar'}，也就是第一个"//"后,'/'前解析为host
    // 如配置为 publicPath: "/assets/"将会得到下面的结果:
    /*
     Url {
      protocol: null,
      slashes: null,
      auth: null,
      host: null,
      port: null,
      hostname: null,
      hash: null,
      search: null,
      query: null,
      pathname: '/assets/
      path: '/assets/',
      href: '/assets/' 
     }
     */
    var localPrefix = urlParse(publicPath || "/", false, true);
    var urlObject = urlParse(url);
    //URL是http请求的真实路径,如http://localhost:1337/hello/world，那么req.url得到的就是/hello/world
    // publicPath has the hostname that is not the same as request url's, should fail
    // 访问的url的hostname和publicPath中配置的host不一致，直接返回。这只有在publicPath是绝对URL的情况下出现
    if(localPrefix.hostname !== null && urlObject.hostname !== null &&
        localPrefix.hostname !== urlObject.hostname) {
        return false;
    }
    // publicPath is not in url, so it should fail
    // publicPath和req.url必须一样
    if(publicPath && localPrefix.hostname === urlObject.hostname && url.indexOf(publicPath) !== 0) {
        return false;
    }
    // strip localPrefix from the start of url
    // 如果url中的pathname和publicPath一致，那么请求成功，文件名为urlObject中除去publicPath那一部分的结果
    // 如上面/hello/world表示req.url，而且publicPath为/hello/那么得到的文件名就是world
    if(urlObject.pathname.indexOf(localPrefix.pathname) === 0) {
        filename = urlObject.pathname.substr(localPrefix.pathname.length);
    }

    if(!urlObject.hostname && localPrefix.hostname &&
        url.indexOf(localPrefix.path) !== 0) {
        return false;
    }
    // and if not match, use outputPath as filename
    //如果有文件名那么从output.path中获取该文件，文件名为我们获取到的文件名。否则返回我们的outputPath
    //也就是说：如果没有filename那么我们直接获取到我们的output.path这个目录
    return filename ? pathJoin(outputPath, filename) : outputPath;
}
module.exports = getFilenameFromUrl;
```

上面这个从URL到路径的转化就是通过webpack-dev-middleware来完成的。


参考资料:

[webpack-dev-middlware](https://github.com/liangklfang/webpack-dev-middleware)

[wcf编译工具开发](https://github.com/liangklfangl/wcf)