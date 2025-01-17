<h1 id="5-3-编写-loader">5-3 编写 Loader</h1>
<p>Loader 就像是一个翻译员，能把源文件经过转化后输出新的结果，并且一个文件还可以链式的经过多个翻译员翻译。</p>
<p>以处理 SCSS 文件为例：</p>
<ol>
<li>SCSS 源代码会先交给 sass-loader 把 SCSS 转换成 CSS；</li>
<li>把 sass-loader 输出的 CSS 交给 css-loader 处理，找出 CSS 中依赖的资源、压缩 CSS 等；</li>
<li>把 css-loader 输出的 CSS 交给 style-loader 处理，转换成通过脚本加载的  JavaScript 代码；</li>
</ol>
<p>可以看出以上的处理过程需要有顺序的链式执行，先 sass-loader 再 css-loader 再 style-loader。
以上处理的 Webpack 相关配置如下：</p>
<pre><code class="lang-js"><span class="hljs-built_in">module</span>.exports = {
  <span class="hljs-built_in">module</span>: {
    rules: [
      {
        <span class="hljs-comment">// 增加对 SCSS 文件的支持</span>
        test: <span class="hljs-regexp">/\.scss/</span>,
        <span class="hljs-comment">// SCSS 文件的处理顺序为先 sass-loader 再 css-loader 再 style-loader</span>
        use: [
          <span class="hljs-string">&apos;style-loader&apos;</span>,
          {
            loader:<span class="hljs-string">&apos;css-loader&apos;</span>,
            <span class="hljs-comment">// 给 css-loader 传入配置项</span>
            options:{
              minimize:<span class="hljs-literal">true</span>, 
            }
          },
          <span class="hljs-string">&apos;sass-loader&apos;</span>],
      },
    ]
  },
};
</code></pre>
<h2 id="loader-的职责">Loader 的职责</h2>
<p>由上面的例子可以看出：一个 Loader 的职责是单一的，只需要完成一种转换。
如果一个源文件需要经历多步转换才能正常使用，就通过多个 Loader 去转换。
在调用多个 Loader 去转换一个文件时，每个 Loader 会链式的顺序执行，
第一个 Loader 将会拿到需处理的原内容，上一个 Loader 处理后的结果会传给下一个接着处理，最后的 Loader 将处理后的最终结果返回给 Webpack。</p>
<p>所以，在你开发一个 Loader 时，请保持其职责的单一性，你只需关心输入和输出。</p>
<h2 id="loader-基础">Loader 基础</h2>
<p>由于 Webpack 是运行在 Node.js 之上的，一个 Loader 其实就是一个 Node.js 模块，这个模块需要导出一个函数。
这个导出的函数的工作就是获得处理前的原内容，对原内容执行处理后，返回处理后的内容。</p>
<p>一个最简单的 Loader 的源码如下：</p>
<pre><code class="lang-js"><span class="hljs-built_in">module</span>.exports = <span class="hljs-function"><span class="hljs-keyword">function</span>(<span class="hljs-params">source</span>) </span>{
  <span class="hljs-comment">// source 为 compiler 传递给 Loader 的一个文件的原内容</span>
  <span class="hljs-comment">// 该函数需要返回处理后的内容，这里简单起见，直接把原内容返回了，相当于该 Loader 没有做任何转换</span>
  <span class="hljs-keyword">return</span> source;
};
</code></pre>
<p>由于 Loader 运行在 Node.js 中，你可以调用任何 Node.js 自带的 API，或者安装第三方模块进行调用：</p>
<pre><code class="lang-js"><span class="hljs-keyword">const</span> sass = <span class="hljs-built_in">require</span>(<span class="hljs-string">&apos;node-sass&apos;</span>);
<span class="hljs-built_in">module</span>.exports = <span class="hljs-function"><span class="hljs-keyword">function</span>(<span class="hljs-params">source</span>) </span>{
  <span class="hljs-keyword">return</span> sass(source);
};
</code></pre>
<h2 id="loader-进阶">Loader 进阶</h2>
<p>以上只是个最简单的 Loader，Webpack 还提供一些 API 供 Loader 调用，下面来一一介绍。</p>
<h3 id="获得-loader-的-options">获得 Loader 的 options</h3>
<p>在最上面处理 SCSS 文件的 Webpack 配置中，给 css-loader 传了 options 参数，以控制 css-loader。
如何在自己编写的 Loader 中获取到用户传入的 options 呢？需要这样做：</p>
<pre><code class="lang-js"><span class="hljs-keyword">const</span> loaderUtils = <span class="hljs-built_in">require</span>(<span class="hljs-string">&apos;loader-utils&apos;</span>);
<span class="hljs-built_in">module</span>.exports = <span class="hljs-function"><span class="hljs-keyword">function</span>(<span class="hljs-params">source</span>) </span>{
  <span class="hljs-comment">// 获取到用户给当前 Loader 传入的 options</span>
  <span class="hljs-keyword">const</span> options = loaderUtils.getOptions(<span class="hljs-keyword">this</span>);
  <span class="hljs-keyword">return</span> source;
};
</code></pre>
<h3 id="返回其它结果">返回其它结果</h3>
<p>上面的 Loader 都只是返回了原内容转换后的内容，但有些场景下还需要返回除了内容之外的东西。</p>
<p>例如以用 babel-loader 转换 ES6 代码为例，它还需要输出转换后的 ES5 代码对应的 Source Map，以方便调试源码。
为了把 Source Map 也一起随着 ES5 代码返回给 Webpack，可以这样写：</p>
<pre><code class="lang-js"><span class="hljs-built_in">module</span>.exports = <span class="hljs-function"><span class="hljs-keyword">function</span>(<span class="hljs-params">source</span>) </span>{
  <span class="hljs-comment">// 通过 this.callback 告诉 Webpack 返回的结果</span>
  <span class="hljs-keyword">this</span>.callback(<span class="hljs-literal">null</span>, source, sourceMaps);
  <span class="hljs-comment">// 当你使用 this.callback 返回内容时，该 Loader 必须返回 undefined，</span>
  <span class="hljs-comment">// 以让 Webpack 知道该 Loader 返回的结果在 this.callback 中，而不是 return 中 </span>
  <span class="hljs-keyword">return</span>;
};
</code></pre>
<p>其中的 <code>this.callback</code> 是 Webpack 给 Loader 注入的 API，以方便 Loader 和 Webpack 之间通信。
<code>this.callback</code> 的详细使用方法如下：</p>
<pre><code class="lang-js"><span class="hljs-keyword">this</span>.callback(
    <span class="hljs-comment">// 当无法转换原内容时，给 Webpack 返回一个 Error</span>
    err: <span class="hljs-built_in">Error</span> | <span class="hljs-literal">null</span>,
    <span class="hljs-comment">// 原内容转换后的内容</span>
    content: string | Buffer,
    <span class="hljs-comment">// 用于把转换后的内容得出原内容的 Source Map，方便调试</span>
    sourceMap?: SourceMap,
    <span class="hljs-comment">// 如果本次转换为原内容生成了 AST 语法树，可以把这个 AST 返回，</span>
    <span class="hljs-comment">// 以方便之后需要 AST 的 Loader 复用该 AST，以避免重复生成 AST，提升性能</span>
    abstractSyntaxTree?: AST
);
</code></pre>
<blockquote>
<p>Source Map 的生成很耗时，通常在开发环境下才会生成 Source Map，其它环境下不用生成，以加速构建。
为此 Webpack 为 Loader 提供了 <code>this.sourceMap</code> API 去告诉 Loader 当前构建环境下用户是否需要 Source Map。
如果你编写的 Loader 会生成 Source Map，请考虑到这点。</p>
</blockquote>
<h3 id="同步与异步">同步与异步</h3>
<p>Loader 有同步和异步之分，上面介绍的 Loader 都是同步的 Loader，因为它们的转换流程都是同步的，转换完成后再返回结果。
但在有些场景下转换的步骤只能是异步完成的，例如你需要通过网络请求才能得出结果，如果采用同步的方式网络请求就会阻塞整个构建，导致构建非常缓慢。</p>
<p>在转换步骤是异步时，你可以这样：</p>
<pre><code class="lang-js"><span class="hljs-built_in">module</span>.exports = <span class="hljs-function"><span class="hljs-keyword">function</span>(<span class="hljs-params">source</span>) </span>{
    <span class="hljs-comment">// 告诉 Webpack 本次转换是异步的，Loader 会在 callback 中回调结果</span>
    <span class="hljs-keyword">var</span> callback = <span class="hljs-keyword">this</span>.async();
    someAsyncOperation(source, <span class="hljs-function"><span class="hljs-keyword">function</span>(<span class="hljs-params">err, result, sourceMaps, ast</span>) </span>{
        <span class="hljs-comment">// 通过 callback 返回异步执行后的结果</span>
        callback(err, result, sourceMaps, ast);
    });
};
</code></pre>
<h3 id="处理二进制数据">处理二进制数据</h3>
<p>在默认的情况下，Webpack 传给 Loader 的原内容都是 UTF-8 格式编码的字符串。
但有些场景下 Loader 不是处理文本文件，而是处理二进制文件，例如 file-loader，就需要 Webpack 给 Loader 传入二进制格式的数据。
为此，你需要这样编写 Loader：</p>
<pre><code class="lang-js"><span class="hljs-built_in">module</span>.exports = <span class="hljs-function"><span class="hljs-keyword">function</span>(<span class="hljs-params">source</span>) </span>{
    <span class="hljs-comment">// 在 exports.raw === true 时，Webpack 传给 Loader 的 source 是 Buffer 类型的</span>
    source <span class="hljs-keyword">instanceof</span> Buffer === <span class="hljs-literal">true</span>;
    <span class="hljs-comment">// Loader 返回的类型也可以是 Buffer 类型的</span>
    <span class="hljs-comment">// 在 exports.raw !== true 时，Loader 也可以返回 Buffer 类型的结果</span>
    <span class="hljs-keyword">return</span> source;
};
<span class="hljs-comment">// 通过 exports.raw 属性告诉 Webpack 该 Loader 是否需要二进制数据 </span>
<span class="hljs-built_in">module</span>.exports.raw = <span class="hljs-literal">true</span>;
</code></pre>
<p>以上代码中最关键的代码是最后一行 <code>module.exports.raw = true;</code>，没有该行 Loader 只能拿到字符串。</p>
<h3 id="缓存加速">缓存加速</h3>
<p>在有些情况下，有些转换操作需要大量计算非常耗时，如果每次构建都重新执行重复的转换操作，构建将会变得非常缓慢。
为此，Webpack 会默认缓存所有 Loader 的处理结果，也就是说在需要被处理的文件或者其依赖的文件没有发生变化时，
是不会重新调用对应的 Loader 去执行转换操作的。</p>
<p>如果你想让 Webpack 不缓存该 Loader 的处理结果，可以这样：</p>
<pre><code class="lang-js"><span class="hljs-built_in">module</span>.exports = <span class="hljs-function"><span class="hljs-keyword">function</span>(<span class="hljs-params">source</span>) </span>{
  <span class="hljs-comment">// 关闭该 Loader 的缓存功能</span>
  <span class="hljs-keyword">this</span>.cacheable(<span class="hljs-literal">false</span>);
  <span class="hljs-keyword">return</span> source;
};
</code></pre>
<h2 id="其它-loader-api">其它 Loader API</h2>
<p>除了以上提到的在 Loader 中能调用的 Webpack API 外，还存在以下常用 API：</p>
<ul>
<li><p><code>this.context</code>：当前处理文件的所在目录，假如当前 Loader 处理的文件是 <code>/src/main.js</code>，则 <code>this.context</code> 就等于 <code>/src</code>。</p>
</li>
<li><p><code>this.resource</code>：当前处理文件的完整请求路径，包括 querystring，例如 <code>/src/main.js?name=1</code>。</p>
</li>
<li><p><code>this.resourcePath</code>：当前处理文件的路径，例如 <code>/src/main.js</code>。</p>
</li>
<li><p><code>this.resourceQuery</code>：当前处理文件的 querystring。</p>
</li>
<li><p><code>this.target</code>：等于 Webpack 配置中的 Target，详情见 <a href="../2配置/2-7其它配置项.html#Target">2-7其它配置项-Target</a>。</p>
</li>
<li><p><code>this.loadModule</code>：当 Loader 在处理一个文件时，如果依赖其它文件的处理结果才能得出当前文件的结果时，
就可以通过 <code>this.loadModule(request: string, callback: function(err, source, sourceMap, module))</code> 去获得 <code>request</code> 对应文件的处理结果。</p>
</li>
<li><p><code>this.resolve</code>：像 <code>require</code> 语句一样获得指定文件的完整路径，使用方法为 <code>resolve(context: string, request: string, callback: function(err, result: string))</code>。</p>
</li>
<li><p><code>this.addDependency</code>：给当前处理文件添加其依赖的文件，以便再其依赖的文件发生变化时，会重新调用 Loader 处理该文件。使用方法为 <code>addDependency(file: string)</code>。</p>
</li>
<li><p><code>this.addContextDependency</code>：和 <code>addDependency</code> 类似，但 <code>addContextDependency</code> 是把整个目录加入到当前正在处理文件的依赖中。使用方法为 <code>addContextDependency(directory: string)</code>。</p>
</li>
<li><p><code>this.clearDependencies</code>：清除当前正在处理文件的所有依赖，使用方法为 <code>clearDependencies()</code>。</p>
</li>
<li><p><code>this.emitFile</code>：输出一个文件，使用方法为 <code>emitFile(name: string, content: Buffer|string, sourceMap: {...})</code>。</p>
</li>
</ul>
<p>其它没有提到的 API 可以去 <a href="https://webpack.js.org/api/loaders/" target="_blank">Webpack 官网</a> 查看。 </p>
<h2 id="加载本地-loader">加载本地 Loader</h2>
<p>在开发 Loader 的过程中，为了测试编写的 Loader 是否能正常工作，需要把它配置到 Webpack 中后，才可能会调用该 Loader。
在前面的章节中，使用的 Loader 都是通过 Npm 安装的，要使用 Loader 时会直接使用 Loader 的名称，代码如下：</p>
<pre><code class="lang-js"><span class="hljs-built_in">module</span>.exports = {
  <span class="hljs-built_in">module</span>: {
    rules: [
      {
        test: <span class="hljs-regexp">/\.css/</span>,
        use: [<span class="hljs-string">&apos;style-loader&apos;</span>],
      },
    ]
  },
};
</code></pre>
<p>如果还采取以上的方法去使用本地开发的 Loader 将会很麻烦，因为你需要确保编写的 Loader 的源码是在 <code>node_modules</code> 目录下。
为此你需要先把编写的 Loader 发布到 Npm 仓库后再安装到本地项目使用。</p>
<p>解决以上问题的便捷方法有两种，分别如下：</p>
<h4 id="npm-link">Npm link</h4>
<p>Npm link 专门用于开发和调试本地 Npm 模块，能做到在不发布模块的情况下，把本地的一个正在开发的模块的源码链接到项目的 <code>node_modules</code> 目录下，让项目可以直接使用本地的 Npm 模块。
由于是通过软链接的方式实现的，编辑了本地的 Npm 模块代码，在项目中也能使用到编辑后的代码。</p>
<p>完成 Npm link 的步骤如下：</p>
<ol>
<li>确保正在开发的本地 Npm 模块（也就是正在开发的 Loader）的 <code>package.json</code> 已经正确配置好；</li>
<li>在本地 Npm 模块根目录下执行 <code>npm link</code>，把本地模块注册到全局；</li>
<li>在项目根目录下执行 <code>npm link loader-name</code>，把第2步注册到全局的本地 Npm 模块链接到项目的 <code>node_moduels</code> 下，其中的 <code>loader-name</code> 是指在第1步中的 <code>package.json</code> 文件中配置的模块名称。</li>
</ol>
<p>链接好 Loader 到项目后你就可以像使用一个真正的 Npm 模块一样使用本地的 Loader 了。</p>
<h4 id="resolveloader">ResolveLoader</h4>
<p>在 <a href="../2配置/2-7其它配置项.html#ResolveLoader">2-7其它配置项</a> 中曾介绍过 ResolveLoader 用于配置 Webpack 如何寻找 Loader。
默认情况下只会去 <code>node_modules</code> 目录下寻找，为了让 Webpack 加载放在本地项目中的 Loader 需要修改 <code>resolveLoader.modules</code>。</p>
<p>假如本地的 Loader 在项目目录中的 <code>./loaders/loader-name</code> 中，则需要如下配置：</p>
<pre><code class="lang-js"><span class="hljs-built_in">module</span>.exports = {
  resolveLoader:{
    <span class="hljs-comment">// 去哪些目录下寻找 Loader，有先后顺序之分</span>
    modules: [<span class="hljs-string">&apos;node_modules&apos;</span>,<span class="hljs-string">&apos;./loaders/&apos;</span>],
  }
}
</code></pre>
<p>加上以上配置后， Webpack 会先去 <code>node_modules</code> 项目下寻找 Loader，如果找不到，会再去 <code>./loaders/</code> 目录下寻找。</p>
<h2 id="实战">实战</h2>
<p>上面讲了许多理论，接下来从实际出发，来编写一个解决实际问题的 Loader。</p>
<p>该 Loader 名叫 comment-require-loader，作用是把 JavaScript 代码中的注释语法</p>
<pre><code class="lang-js"><span class="hljs-comment">// @require &apos;../style/index.css&apos;</span>
</code></pre>
<p>转换成</p>
<pre><code class="lang-js"><span class="hljs-built_in">require</span>(<span class="hljs-string">&apos;../style/index.css&apos;</span>);
</code></pre>
<p>该 Loader 的使用场景是去正确加载针对 <a href="http://fis.baidu.com/fis3/docs/user-dev/require.html" target="_blank">Fis3</a> 编写的 JavaScript，这些 JavaScript 中存在通过注释的方式加载依赖的 CSS 文件。</p>
<p>该 Loader 的使用方法如下：</p>
<pre><code class="lang-js"><span class="hljs-built_in">module</span>.exports = {
  <span class="hljs-built_in">module</span>: {
    rules: [
      {
        test: <span class="hljs-regexp">/\.js$/</span>,
        use: [<span class="hljs-string">&apos;comment-require-loader&apos;</span>],
        <span class="hljs-comment">// 针对采用了 fis3 CSS 导入语法的 JavaScript 文件通过 comment-require-loader 去转换 </span>
        include: [path.resolve(__dirname, <span class="hljs-string">&apos;node_modules/imui&apos;</span>)]
      }
    ]
  }
};
</code></pre>
<p>该 Loader 的实现非常简单，完整代码如下：</p>
<pre><code class="lang-js"><span class="hljs-function"><span class="hljs-keyword">function</span> <span class="hljs-title">replace</span>(<span class="hljs-params">source</span>) </span>{
    <span class="hljs-comment">// 使用正则把 // @require &apos;../style/index.css&apos; 转换成 require(&apos;../style/index.css&apos;);  </span>
    <span class="hljs-keyword">return</span> source.replace(<span class="hljs-regexp">/(\/\/ *@require) +((&apos;|&quot;).+(&apos;|&quot;)).*/</span>, <span class="hljs-string">&apos;require($2);&apos;</span>);
}

<span class="hljs-built_in">module</span>.exports = <span class="hljs-function"><span class="hljs-keyword">function</span> (<span class="hljs-params">content</span>) </span>{
    <span class="hljs-keyword">return</span> replace(content);
};
</code></pre>
<blockquote>
<p>本实例<a href="https://github.com/gwuhaolin/comment-require-loader" target="_blank">提供项目完整代码</a></p>
</blockquote>

                                
                                